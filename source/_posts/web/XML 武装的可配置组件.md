title: "XML 武装的可配置组件"
date: 2017-01-10 21:11:11
categories: Web
---

本篇主要总结「XML 解析并生成组件」相关功能。
<!-- more -->

### 一、模块重构

在一期需求开发中：

 - 后端返回的所有数据直接挂在 `Regular.data` 的一个子对象中；
 - 一个页面的所有逻辑全部写在一个 html 中。

简单来讲，刚开始的新手村任务完成得拓展性太差。
因此二期开发的初期，代码在拓展时逻辑非常难捋，一团乱麻。

这时候必须进行重构，重构流程分三步：

 1. **模块划分**

基于交互稿，依据页面逻辑划分模块。

 2. **数据划分**

 - 根据页面模块划分，将返回数据分配到不同的模块对应的对象中

```js
data.baseInfo = {};
data.rejectInfo = {};

data.templateList = [];
data.templateInfo = {};

data.materialList = [];
data.materialInfo = {};
```

- 默认值设定

在一期时，由于直接将返回数据存入 `Regular.data` 中，且后端返回的数据不可控，因此在页面模板中出现了大量的存在判断：

```html
<p>{exampleInfo ? exampleInfo.reason : '无'}<p>
```

但其实并不应该在模板中做这样大量的存在判断，而是在数据分配阶段，就对所有返回数据进行统一的默认值设置：

```js
data.baseInfo = {
    pageName: res.pageName || '',
    deliveryors: res.deliveryors || [],
    // ...其它数据
};
```

 - 统一的状态控制

页面需要进行「查看」/「编辑」、「自己上传素材」/「根据素材 ID 上传」等的状态切换。
这部分数据都是页面配置数据，统一保存到 config 对象中，方便各组件使用，也易于拓展。

```js
data.config = {
    isEdit: 0,
    uploadType: 0,
    // ...其它配置数据
};
```

 - 使用局部变量

① 页面渲染时或后续操作时需要使用的数据才存储到 `data` 中，不要把所有数据都挂到 `data` 上；

② 用于数据的初始化处理的高频数据，使用局部变量缓存，可以保持 `data` 干净；

③ 作用域链较长的数据，使用局部变量缓存，减少对象属性的查找次数，提升代码可读性。

 3. **组件封装**

完成「模块划分」和「数据划分」后，就可以把原来各个模块的 **页面逻辑 (html)** 和 **操作逻辑 (function)** 抽取出来，封装成各个模块组件。

```html
<form class="m-form m-material">
    <!-- 时间和说明 -->
    <deliveryInfo
        config={config}
        baseInfo={baseInfo}
    ></deliveryInfo>

    <!-- 样式表 -->
    <styleList
        config={config}
        templateList={templateList}
        templateInfo={templateInfo}
        on-changeStyle={this.changeItem($event)}
    ></styleList>

    <!-- 上传形式选择模块 -->
    <uploadWay
        config={config}
        templateInfo={templateInfo}
        materialInfo={materialInfo}
    ></uploadWay>

    <!-- 素材模块 -->
    <material
        config={config}
        templateInfo={templateInfo}
        materialInfo={materialInfo}
    ></material>
</form>
```

这时候页面组件更多作为一个纯组件容器，模块内部交互操作由模块组件自行负责。

其中需求核心 —— 涉及「XML 解析」的素材逻辑，就可以交给 material 素材组件独立完成，也方便其他页面复用该模块。

---

### 二、核心需求描述

二期需求的核心是涉及「XML 解析」的组件可配置功能。

> **为什么要用 XML 这种结构来实现组件可配置？**
>
> 答：因为 XML 有 XSD（类似 HTML 的 DTD）可以进行格式校验，确保 XML 格式正确。

该系统的 XML 文档中总共可能有以下十种素材：

| 我 | 们 | 是 | 素 | 材 |
| :-: | :-: | :-: | :-: | :-: |
| 1. 图片 | 2. 链接 | 3. 文本 | 4. 枚举 | 5. 数量 |
| 6. 商品 | 7. 专辑 | 8. 品牌 | 9. 优惠券 | 10. 列表 |

> 注：列表素材中还可能包含其它九种素材的任意几种。

以下面的 XML 文档内容为例：

```xml
<image necessary="true" key="avatar" desc="一张大图" format="jpg" width="960" height="210" />
<itemList necessary="true" key="itemList" desc="图片列表" minLimit="1" maxLimit="8" >
    <goods necessary="true" key="goodsId" isInStock="false" desc="商品ID" explain="请填写母婴类目商品" />
    <enum necessary="true" key="type" desc="下拉选择框" enum="1|普通,2|签到,3|分享" explain="选择框描述字段" />
    <number necessary="true" key="number" desc="商品数量" explain="商品数量" min="1" max="100" />
</itemList>
```

上述文档中素材的结构为：

 1. 「图片」素材；
 2. 「列表」素材：
    1. 「商品」素材；
    2. 「枚举」素材；
    3. 「数量」素材。

基于 XML 中存放的格式定义，XML 有**三大重任**：

 1. 生成空白的素材数据，用于用户填写和提交：

```js
var materialInfo = {
    avatar: '',
    itemList: [
        {
            goodsId: '',
            type: '',
            number: ''
        },
        // 其他项，与第一项结构相同
    ]
};
```

 2. 根据 XML 结构生成素材组件（结构复杂程度决定素材组件的复杂程度）：

![component][component]

 3. 为素材组件的校验规则提供信息，如「是否必填」、「边界值」、「宽高」等限制：

![checker][checker]

---

### 三、XML 解析

XML 解析流程分为以下两步：

 1. 将接口返回的 XML 字符串转换为 XML 文档

这里主要利用的是 [DOMParser][DOMParser] 对象和它的 `parseFromString` 方法：

```js
// 举例用的 XML 字符串
var xmlString = '<?xml version=\"1.0\" encoding=\"UTF-8\"?><module necessary=\"true\" desc=\"大图一拖八模块\" thumbnail=\"imageUrl\" preview=\"http://shared.ydstatic.com/r/2.0/p/fanyi-logo-s.png\"><image necessary=\"true\" key=\"avatar\" desc=\"一张大图\" format=\"jpg\" width=\"960\" height=\"210\"/><itemList necessary=\"true\" key=\"itemList\" desc=\"图片列表\" minLimit=\"1\" maxLimit=\"8\"><module className=\"com.netease.kaola.Item.class\" desc=\"\"><goods necessary=\"true\" key=\"goodsId\" isInStock=\"false\" desc=\"商品ID\" explain=\"请填写母婴类目商品\"/><enum necessary=\"true\" key=\"type\" desc=\"下拉选择框\" enum=\"1|普通,2|签到,3|分享\" explain=\"选择框描述字段\" /><number necessary=\"true\" key=\"number\" desc=\"商品数量\" explain=\"商品数量\" min=\"1\" max=\"100\" /></module></itemList></module>';

var parser = new DOMParser();
var doc = parser.parseFromString(xmlString, "application/xml");
```

处理后生成如下的 XML 文档：

![XML Document][XML Document]

> 注：由 [caniuse][caniuse] 可知，`DOMParser` 在现代浏览器中兼容性良好。

 2. 将 XML 文档转换为 JSON 对象

对 XML 文档进行「**深度优先先序遍历**」，对每个元素节点（elementNode，`nodeType` 为 1）进行处理，生成一个 JSON 对象。

以 image 元素节点的处理为例：

```xml
<image necessary="true" key="avatar" desc="一张大图" format="jpg" width="960" height="210"/>
```

处理后（关键词：`nodeName`、`attributes`）生成的 JSON 对象长这样：

```js
avatar: {
    $attr: {
        necessary: true,
        key: 'avatar',
        desc: '一张大图',
        format: 'jpg',
        width: '960',
        height: '210'
    },
    $target: 'image'
}
```

可以看出：

 - 属性名：元素节点 `attributes` 中的 `key` 值，与素材数据的结构对应；
 - `$attr`：格式信息，存放着该素材所有的格式限制和信息提示；
 - `$target`：类型说明，利用该属性识别素材的类型。

经过一次完整遍历后生成的终极 JSON 对象：

```js
// 数据不完全展开
var scheme = {
    $attr: {},
    avatar: {
        $attr: {},
        $target: 'image'
    },
    itemList: {
        $attr: {},
        $target: 'itemList',
        info: [
            {
                goodsId: {
                    $attr: {},
                    $target: "goods"
                },
                number: {
                    $attr: {},
                    $target: "number"
                },
                type: {
                    $attr: {},
                    $target: "enum"
                }
            },
            // 其它项，与第一项格式相同
        ]
    }
}
```

> 注：XML 解析时需要注意 Boolean 类型的值转换。
>
> 若转换成 `'false'`，在逻辑判断时造成的错误可能就比较难发现。

---

### 四、可配置组件

 1. 元组件编写

编写十种素材组件，统一放在素材组件的 `components` 文件夹中。

注意：组件的格式信息统一为 `format`，配置信息统一为 `config`，方便第二步模板配置时动态替换。

 2. 模板配置

由于「XML 解析出来的格式信息」决定「最终生成的素材组件的结构」，所以接下来需要完成模板配置。

通过 templates 模板文件实现组件模板管理：

```js
module.exports = {
    itemList: '<itemlist scheme={scheme.${name}} list={info.${name}} config={config}></itemlist>',
    text: '<text format={scheme.${name}.$attr} text={info.${name}} config={config}></text>',
    image: '<image format={scheme.${name}.$attr} image={info.${name}} config={config}></image>',
    link: '<url format={scheme.${name}.$attr} url={info.${name}} config={config}></url>',
    brand: '<brand format={scheme.${name}.$attr} brandId={info.${name}} config={config}></brand>',
    goods: '<goods format={scheme.${name}.$attr} goodsId={info.${name}} config={config}></goods>',
    coupon: '<coupon format={scheme.${name}.$attr} couponId={info.${name}} config={config}></coupon>',
    album: '<album format={scheme.${name}.$attr} albumId={info.${name}} config={config}></album>',
    enum: '<downbox format={scheme.${name}.$attr} type={info.${name}} config={config}></downbox>',
    number: '<number format={scheme.${name}.$attr} num={info.${name}} config={config}></number>'
};
```

这里借鉴 ES6 中模板字符串的变量定义方式来做变量替换。

继续用前面的例子，处理 `scheme`：

```js
/**
 * 根据 xml 转换的 json 结构生成对应的 html 模板字符串
 * @param  {[type]} scheme         [定义素材数据的格式]
 * @param  {[type]} templates      [元组件类型：对应 html 模板 |　映射关系]
 * @return {[type]} templateString [生成 html 模板字符串]
 */
var generateTemplate = function(scheme, templates) {
    var templateList = [], realTemplate = '';
    for (var item in scheme) {
        if (scheme.hasOwnProperty(item) && scheme[item].hasOwnProperty('$target')) {
            templateItem = templates[scheme[item].$target].replace(
                /\${name}/g, item);
            templateList.push(templateItem);
        }
    }
    return templateList.join('');
};
```

前文例子生成的 `scheme` JSON 对象利用该函数处理的流程是：

 1. 根据素材 `$target` 找到对应的模板 `image` 和 `itemlist`；
 2. 将 `${name}` 替换为 `avatar` 和 `itemList`；
 3. 拼接字符串并返回：

```js
'<image format={scheme.avatar.$attr} image={info.avatar} config={config}></image>'
'<itemlist scheme={scheme.itemList} list={info.itemList} config={config}></itemlist>'
// 列表组件是一个子素材组件，所以 itemList 的插值是 scheme 而不是 format
```

itemList 列表组件内部的处理方式相同:

```js
'<goods format={scheme.goodsId.$attr} goodsId={info.goodsId} config={config}></goods>'
'<downbox format={scheme.type.$attr} type={info.type} config={config}></downbox>'
'<number format={scheme.number.$attr} num={info.number} config={config}></number>'
```

 3. 模板引入

到这里，直接在素材组件的 html 文件中引入模板字符串即可：

```html
{#include templateString}
```

最终页面生成的素材模块为：

![component][component]

---

### 六、校验

Regular UI 自带的表单验证功能只适用于平级结构，并不适合组件层级较深的情况。
而系统其他地方可以平稳使用该验证功能，再引入其它的验证库也没有必要。

由于每个素材的 `format` 中存储着数据验证时需要的验证信息，所以可以简单利用 `format` 实现数据验证：

 1. 在每个素材对应的 format 中添加 `isCorrect` 属性，作为数据验证是否通过的判断依据；

 2. 封装数据验证方法，一种素材对应一种验证方法，在离开当前素材组件焦点时进行元组件的数据验证；

 3. 封装数据验证方法，在对完整素材数据进行提交时遍历完整的素材数据，依次执行素材各自的验证方法实现验证。

完成验证，整个素材模块也就真正完成。

---

### 七、其他

 - 统一 API

系统用的组件都继承于一个包含 `$request` 方法（用于发送异步请求）的组件。
如果一个组件模块有多处异步请求操作，那么在一份 JS 文件中要对它们进行统一修改时，定位和修改都不那么优雅。

因此自己把所有的异步请求函数都都抽取出来，放在每个组件的 `assets` 子文件夹中的 `API.js` 中统一管理，并以 Promise 方式返回。

例如：

```js
module.exports = {
    /**
     * 获取记录列表信息
     * @param  {[object]} queryInfo [获取记录列表需要的查询参数]
     * @return {[array]}           [查询到符合条件的查询列表数据]
     */
    gettingRecordList: function(queryInfo) {
        var reqInfo = {
            url: '/record?' + ut.param(queryInfo),
            opts: {
                method: 'get',
                timeout: 8000
            }
        };
        return new Promise(function(resolve, reject) {
            $request(reqInfo.url, reqInfo.opts).then(function(res) {
                if(!!res && res.code === 200) {
                    resolve(res.list || []);
                } else {
                    reject('响应数据错误，无法获取列表信息！');
                }
            }).catch(function(err) {
                reject('无法获取列表信息：' + (err || '无信息'));
            });
        });
    }
};
```

在 API 文件中额外封装 `$request` 函数并不合理，改为以 mixin 的方式将方法引入各组件中。

 - 统一自定义错误

由上述返回的 Promise 对象，在实际调用的时候可以：

```js
API.gettingRecordList(queryInfo).then(function(res) {
    if (wrong) {
        throw new Error('出现某些错误！');
    }
    // ...some code
}).catch(function(err) {
    Notify.warning("获取失败：" + (err || '无信息'));
    console.warn("获取失败：" + (err || '无信息'));
});
```

Promise 的 reject 中返回的错误信息，及处理数据时抛出的错误，由 catch 函数统一输出。

 - 中英文区分

在某个输入框中需要区别中英文字符长度，中文算两个字符，因为中文占宽较大。

既然原因是占宽问题，那么最合适的处理方式应该是全半角区分。

这时直接利用正则获取不包含在单字节编码范围内的字符，比做中英文判断更全面：

```js
ASCII: [^\x00-\xff]
Unicode: [^\u0000-\u00FF]
```

关于 Unicode 和 ASCII 与 JavaScript 的关系可以看看：

[阮一峰：《字符编码笔记：ASCII，Unicode 和 UTF-8》](coding1)

[阮一峰：《Unicode 与 JavaScript 详解》](coding2)

 - Codewars

[Codewars](codewars) 是一个跟 LeetCode 类似但更低调的在线编程解题网站（都支持 JavaScript）。

「模板配置」部分参考了以前刷过的一道题「[Who likes it?](topic)」的思路。

没事刷刷题其实也蛮好玩的，推荐给大家。

[DOMParser]: https://developer.mozilla.org/en-US/docs/Web/API/DOMParser
[XML Document]: http://aeo.ijarvis.cn/xml/XMLDocument.jpg
[caniuse]: http://caniuse.com/#search=DOMParse
[component]: http://aeo.ijarvis.cn/xml/component.jpg
[checker]: http://aeo.ijarvis.cn/xml/checker.jpg
[coding1]: http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html
[coding2]: http://www.ruanyifeng.com/blog/2014/12/unicode.html
[codewars]: https://www.codewars.com
[topic]: https://www.codewars.com/kata/who-likes-it/train/javascript
[course]: http://open.163.com/movie/2006/8/3/8/M7BC8JMHJ_M7BC8PA38.html