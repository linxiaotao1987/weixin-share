# 微信分享功能：在SPA下，前一个页面禁用分享导致下一个页面开启分享后config失败

### 场景和问题描述：

ME的课程列表系列页面采用的是react来做的SPA，路由采用的是哈希值跳转；
入口页面是用户的课程列表页，包括付费的和免费的，产品要求这一页为不可分享；
点击列表项进入课堂列表，由于课堂有付费和免费的区别，所以这一页也不可分享；
点击课堂列表进入课程详情页，产品要求这一页为可分享，分享内容需要自定义。如果是付费课而点击进去的用户没有够买这节课，那么跳转到购买页；
也即是：
用户课程列表页（入口，不可分享，后称A页面） =>课程的课堂列表页（不可分享，后称B页面） => 课堂的详情页（可分享并且分享内容自定义，后称C页面）

实现过程：
根据微信官网说明，在A、B页面componentDidMount的时候监听WeixinJSBridgeReady事件，事件触发再执行wx.hideOptionMenu()。不要使用WeixinJSBridge.call('hideOptionMenu')，虽然也有效果，但是微信会提示这个方法不被支持。

在C页面componentDidMount的时候发起ajax请求，将当前页面的地址作为data；由于使用的是哈希值跳转，而微信会截掉哈希值，所以需要用encodeURIComponent将地址转换，这一步之后还需要后端进行相应处理。注意请求头和type的类型：
```
linAjax({
        type: 'post',
        url: ‘后端给的请求地址’,
        data: {
            "url": encodeURIComponent(location.href)
        },
        timeout: '',
        success: function (data) {

            data = JSON.parse(data);
            if (data.code === 0) {
                wxRegister(data);
            }
        },
        error: function (data) {
            console.log('数据出错');
        },
        setRequestHeader: {
            'Accept': 'application/json',
            'X-Requested-With': 'XMLHttpRequest',
            'content-type': 'application/x-www-form-urlencoded'
        }
    });
```
ajax请求成功后进行wx的config，注意上面的success函数里的wxRegister，这个是自定义函数：
```
function wxRegister(obj) {
        var data = obj.data;
        wx.config({
            debug: false,
            appId: data.appId,
            timestamp: data.timestamp,
            nonceStr: data.nonceStr,
            signature: data.signature,
            jsApiList: ['onMenuShareTimeline', 'onMenuShareAppMessage']
        });
        wx.ready(function () {
            share();
        });
        wx.error(function(res){

        });
    }
```
ready后的share函数是这样的：
```
function share() {
        wx.onMenuShareTimeline({
            title: , // 分享标题
            link: window.location.href, // 分享链接
            imgUrl: '', // 分享图标
        });
        wx.onMenuShareAppMessage({
            title: , // 分享标题
            desc: , // 分享描述
            link: window.location.href, // 分享链接
            imgUrl: '', // 分享图标
            type: '',
            dataUrl: ''
        });
    }
```
做完以上步骤后测试页面，在ios端发现A，B页面禁用分享成功，C页面启用分享失败；
在C页面componentDidMount里开始请求数据之前开启wx.showOptionMenu()，页面开启分享成功，但是config失败。

不要尝试这样干，因为已经被证明无效：
A页面wx.hideOptionMenu()，B页面：wx.showOptionMenu() + wx.hideOptionMenu()，
在AB页面都使用：wx.showOptionMenu() + wx.hideOptionMenu()，C页面依然是config失败

能够正常工作的做法是：
A页面componentDidMount里发起ajax请求，拿到数据后执行wx.config，然后调用share设置。即是把页面当做需要分享的页面来做，执行了一开始那一套程序linAjax（）=》wxRegister(data)=》share（）；最后再调用wx.hideOptionMenu()禁止分享；
B页面同A；
最后到C页面就可以正常设置分享了。

项目需求的问题到此就算解决了，再思考多几步。假设入口页是需要分享的，那会怎么样？
1、如果A页什么也不做，那么A页面可以分享，但是分享内容为默认效果；
B页面执行分享请求等设置再禁用分享，config失败；
同理，C页面也config失败，呵呵哒。
2、A页面执行分享请求等设置,B设置后再禁用，那么C页面能正常工作。

这样搞，如果分享的目标页的路径很深，那么就麻烦啦。在他之前的那些页面都需要用ajax请求一遍，对性能和带宽都是浪费。怎么办？强制页面刷新是一种办法。
虽然可以捕捉到wx.config的error状态，可惜location.href = location.href、location.reload()、location.href = location.href +"?id="+10000*Math.random();都没效。
暂时没有找到微信端刷新页面的js方法。
