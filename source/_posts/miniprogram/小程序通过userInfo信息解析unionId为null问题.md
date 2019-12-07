---
title: 小程序通过userInfo信息解析unionId为null问题
tags: 小程序
categories: 小程序
---

##   简述
最近在开发一款小程序，因人手不足，我就来充数了~ 而我们测试一直在提醒我，今天抽时间看了下，
具体描述和解决过程请戳下面

<!--more-->

##   问题重现

*   由于项目不保证用户关注公众号，所以只能通过解密getUserInfo中的encryptedData获取

*   然后得有两个端的设备（手机，微信开发者工具）；

*   先在微信开发者工具登录成功，然后使用手机登录踢掉，最后在微信开发者工具上重新登录，
问题出现

*   因无法取到unionId导致无法登录

##  排查
*   这边出现问题，抛出的错误不明显，所以我到k8s上查看错误日志，发现日志如下：
```
javax.crypto.BadPaddingException: pad block corrupted
	at org.bouncycastle.jcajce.provider.symmetric.util.BaseBlockCipher$BufferedGenericBlockCipher.doFinal(Unknown Source)
	at org.bouncycastle.jcajce.provider.symmetric.util.BaseBlockCipher.engineDoFinal(Unknown Source)
	at javax.crypto.Cipher.doFinal(Cipher.java:2164)
	at com.galaxy.base.business.impl.wechat.WechatUtil.decryptOfDiyIV(WechatUtil.java:78)
```
单看错误不知道是啥意思，google看是AES加解密错误

*   然后我多试了几次，还好我们的日志很完整，我发现一个很有趣的问题，就是登出以后再登录的时候，
一旦session_key与上一次一样就能够成功，而一旦不一样就无法解密。基本确定是session_key有问题，
然后看了文档《[开放数据校验与解密]("https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/signature.html")》中的如下介绍：
```
会话密钥 session_key 有效性
开发者如果遇到因为 session_key 不正确而校验签名失败或解密失败，请关注下面几个与 session_key 有关的注意事项。

wx.login 调用时，用户的 session_key 可能会被更新而致使旧 session_key 失效（刷新机制存在最短周期，如果同一个用户短时间内多次调用 wx.login，并非每次调用都导致 session_key 刷新）。开发者应该在明确需要重新登录时才调用 wx.login，及时通过 auth.code2Session 接口更新服务器存储的 session_key。
微信不会把 session_key 的有效期告知开发者。我们会根据用户使用小程序的行为对 session_key 进行续期。用户越频繁使用小程序，session_key 有效期越长。
开发者在 session_key 失效时，可以通过重新执行登录流程获取有效的 session_key。使用接口 wx.checkSession可以校验 session_key 是否有效，从而避免小程序反复执行登录流程。
当开发者在实现自定义登录态时，可以考虑以 session_key 有效期作为自身登录态有效期，也可以实现自定义的时效性策略。
```

#   解决

*   原来每次使用wx.login会更新session_key,看了我们的代码确实有问题，我简单写下我们原来的逻辑,
每次都是先获取到encryptedData和iv,再获取login刷新了session_key，然后解析当然就错了~
```
//wxml
<button class='btn-purple' open-type="getUserInfo" bindgetuserinfo="weixinLogin">
        <image src='../../images/common/icon_weixin.png' mode='widthFix'></image>微信授权登录</button>

//js
weixinLogin: function (e) {
    var encryptedData = e.detail.encryptedData;
    var iv = e.detail.iv;

        wx.login({
          success: res => {
                wx.request({
                  url: app.globalData.webUrl + '/api/wechat/v1/getLogin',
                  data: {
                    miniCode: res.code,
                    encryptedData: encryptedData,
                    iv: iv
                  },
                  header: {
                    'content-type': 'application/x-www-form-urlencoded'
                  },
                  method: 'GET',
                  success: function (res) {
                    //success goto index
                  }
          },
          fail: res => {
            this.wetoast.toast({ title: res.err_desc });
          }
        })
}
```

*   解决也很简单，可以在onLoad的时候先获取code，然后去掉错误代码里的wx.login，简化带如下
```
//获取code 在onLoad内调用一下
  getLogin(){
    var that = this;
    wx.login({
      success: res => {
        console.log(res);
        that.setData({code: res.code})
      },
      fail: res => {
        this.wetoast.toast({ title: res.err_desc });
      }
    })
  },

//
weixinLogin: function (e) {
    var that = this;
    var encryptedData = e.detail.encryptedData;
    var iv = e.detail.iv;

    wx.request({
      url: app.globalData.webUrl + '/api/wechat/v1/getLogin',
      data: {
        //这里改一下
        miniCode: that.data.code,
        encryptedData: encryptedData,
        iv: iv
      },
      header: {
        'content-type': 'application/x-www-form-urlencoded'
      },
      method: 'GET',
      success: function (res) {
        //success goto index
  }
}
```