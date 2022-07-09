---
title: 记一个沙雕软件的破解
date: 2022-07-10T02:23:58+08:00
draft: false
---
本文是一篇日记性质的碎碎念,唠叨和啰嗦在所难免,请见谅.

故事是这样的,我突然想要一个能够在ubuntu上同步Google Drive的软件.我需要这个软件能
够监控我磁盘上的文件变化,一旦文件变化了就上传;当然Google Drive上的文件要是发生
了变化,也得能够同步到我的磁盘上.

本来以为[RClone](https://rclone.org/)可以,但是后来发现它只能单向同步,相当于是个
备份工具,不能够满足我的要求.

其实我也不希望搞那种很专业的工具,能有谷歌自己在Windows上做的那个"备份与同步" 那
点功能,其实就可以了.配置最好也不要太麻烦.

所以我找到了[Insync](https://www.insynchq.com/),这是一个支持OneDrive和Google
Drive的同步客户端.并不开源,而且售价有点高,要40刀.

我暂时用不到OneDrive,而且这个钱说实话是不想花的(主要还是实在太贵了.

于是我转头又找到了OverGrive,这是一个南非的网站(/组织?)搞的一个Google Drive客户
端,也不免费,但是只要5刀了,勉强可以接受.

于是我拿出了我办的浦发双币visa卡.这张卡我办完了以后当时拿来尝试绑Google Play没绑
上,所以就一直没用上.不过不用公本费,也没有年金,更没有额度,我只能用支付宝,或者其
他银行卡往里面存钱,之后才能消费美元.

这下终于算是派上了用场.

-----

然后迎接我的就是付款失败.里面的钱明明够啊?为啥呢?

打客服电话问了一下,说是这个商户被判定为有可能是投资目标之类的东西,所以被限制交易
了.啊这(

那没什么办法,我就淘宝上找了个paypal代付.

结果24h过去了,我仍然没有从我的邮降收到我的激活码邮件,我非常确定没有在垃圾桶里.

当然这个软件是有试用版的,有14天的试用期,可能是开发者认为只要14天以内发激活码就行
了么???

不过,由于这个软件存在一些启动过程中的问题,所以我稍微研究了一下它,发现这是一个
python写的东西,于是我用[uncompyle6][==link1==],给它反编译了一下,看到了源码.

不过反编译出来的源码并不能正常运行,存在一些问题,应该是反编译的时候控制流识别出错
了.

但是我却因此能够看到它的激活流程:

``` python
def on_activation_button(widget):
    global licenseDisplay
    global splashProgressBar
    global splashWindow
    if 'ProgressBar' in str(splashGrid.get_child_at(0, 4)):
        Gtk.Container.remove(splashGrid, splashProgressBar)
        splashGrid.attach(splashEntry, 0, 4, 2, 1)
    splashEntry.set_tooltip_text(_('Copy and Paste your') + ' ' + appDisplayName + ' ' + _('Activation code'))
    splashEntry.set_placeholder_text(_('Enter Activation Code'))
    splashLabel3.set_markup('<a href="http://www.thefanclub.co.za/overgrive" >' + _('Get Activation Code') + '</a>')
    splashLabel3.connect('activate-link', openInBrowserLisence, 'http://www.thefanclub.co.za/overgrive')
    activationCode = splashEntry.get_text()
    activationCode = activationCode.strip()
    if activationCode:
        debugPrintAndLog('[SETUP] Activation Code : ' + activationCode)
        if activateLicense(activationCode):
            licenseDisplay = _('License Activated')
            splashWindow.destroy()
            Gtk.main_quit()
        else:
            splashEntry.set_text('')
            splashEntry.set_placeholder_text(_('Activation Code') + ' ' + _('Error'))
```

这里调用了`activateLicense(activationCode)`来验证激活码;

``` python
def activateLicense(entry):
    if entry == signature(email_address):
        get_activation = getAppData(drive_service, 'Activated')
        debugPrintAndLog('[License] Activation : ' + str(get_activation))
        if get_activation == 'Activated':
            return True
    else:
        debugPrintAndLog('[License] Activation Code Incorrect.')
        return False
```

这里可以看到`entry == signature(email_address)`是重点;

``` python
def signature(text):
    message = bytes(text.encode('utf-8'))
    tmp_sec = text + '#overgrive'
    secret = bytes(tmp_sec.encode('utf-8'))
    signatureEncode = base64.b64encode(hmac.new(secret, message, digestmod=(hashlib.sha512)).digest())
    return signatureEncode.decode('utf-8')
```

看到这里我震惊了,这个 Key 的验证居然是完全在本地完成的. 而且明明白白把生成验证
码的方式都告诉我了,那我就不客气了.

``` python
import base64, hmac, hashlib


def signature(text):
    message = bytes(text.encode("utf-8"))
    tmp_sec = text + "#overgrive"
    secret = bytes(tmp_sec.encode("utf-8"))
    signatureEncode = base64.b64encode(
        hmac.new(secret, message, digestmod=(hashlib.sha512)).digest()
    )
    return signatureEncode.decode("utf-8")


email = input("Your Google Drive Account Email: ")
print("Your Activation Code: " + signature(email))
```

完事.

[==link1==]: https://github.com/rocky/python-uncompyle6
