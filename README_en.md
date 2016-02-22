# Public WeChat Python-SDK

Author: [@jeff_kit](http://twitter.com/jeff_kit)
Translator: [@icedwater](http://twitter.com/icedwater)

This SDK supports WeChat messaging protocols for use with public accounts and
enterprise accounts. OAuth authentication is supported too. This file and SDK
assumes that the user already knows the basics of message transmission with a
WeChat account, and is able to find relevant API documentation for the public
and enterprise cases.

## 1. Installation

### pip
	
	pip install wechat


### Source Installation

	git clone git@github.com:jeffkit/wechat.git
	cd wechat
	python setup.py install


## 2. Messaging Framework

This SDK provides a mini-framework for messaging using public accounts, which
may be found in `wechat.official.WxApplication`. This allows use with the web
framework that one may prefer.

`WxApplication` takes care of the well-formedness of requests and messages so
that developers may focus on the business logic.

WxApplication core method:

### WxApplication.process(params, xml, token=None, app_id=None, aes_key=None)

`WxApplication.process` accepts the following parameters:

- `params`: dictionary with the components of the `url` `querystring` from a
   WeChat response. Example: {'nonce': 1232, 'signature': 'xsdfsdfsd'}
- `xml`, the XML content returned via POST by WeChat.
- `token`, a public authentication token (optional).
- `app_id`, the application ID from a public account (optional).
- `aes_key`, the `secret` from a public account (optional).

`process` returns a character string, which may be XML or an `echoStr`.


#### Use Case 1：URL Verification

Once the public WeChat account backend has configured the `URL`, `token`, and
other information correctly, WeChat will issue a `GET` request for the `URL`.
Developers may use `app.process(params, xml=None)` to reply with an `echoStr`
in the callback for this request:

	qs = 'nonce=1221&signature=19selKDJF&timestamp=12312'
	query = dict([q.split('=') for q in qs.split('&')])
	app = YourApplication()
	echo_str = app.process(query, xml=None)
	# Send echo_str back to WeChat
	

#### Use Case 2: Handling ?updates

Users sending a message to a public WeChat account from a public account, the
Wechat server will use the transmitted URL. Developers must perform checks on
every request for validity and message-handling, and this can be handled with
`app.process`:

	qs = 'nonce=1221&signature=19selKDJF&timestamp=12312'
	query = dict([q.split('=') for q in qs.split('&')])
	body = '<xml> ..... </xml>'
	app = YourApplication()
	result = app.process(query, xml=body)
	# Send result back to WeChat


### Inheriting from WxApplication

This sample code using `WxApplication` which returns an uploaded text string:

	from wechat.official import WxApplication, WxTextResponse, WxMusic,\
		WxMusicResponse
	
	class WxApp(WxApplication):
	
    	SECRET_TOKEN = 'test_token'
	    WECHAT_APPID = 'wx1234556'
	    WECHAT_APPSECRET = 'sevcs0j'

    	def on_text(self, text):
        	return WxTextResponse(text.Content, text)


The type and number of required parameters can be found in the WeChat backend
developers pages. If the first three parameters are not set here, they should
be passed to the `process` method when calling it.
	
- `SECRET_TOKEN`: Response token for the WeChat public account
- `APP_ID`: Application ID for a WeChat public account ID
- `ENCODING_AES_KEY`: (optional) A `SECRET`, not needed if the public account
   has not added secure transmission.
- `UNSUPPORT_TXT`: (optional) text response when receiving unsupported data.
- `WELCOME_TXT`: (optional) default text response for new followers.

Next, the `WxApplication` event handlers `on_xxx` need to be implemented. For
various types of received messages, there are appropriate `on_xxx` handlers.

### `on_xxx` handlers


Here are all the `on_xxx` handlers:

- `on_text`: respond to text from the user
- `on_link`: respond to a link from the user
- `on_image`: respond to an image uploaded by the user
- `on_voice`: respond to a recorded audio file from the user
- `on_video`: respond to an uploaded video from the user
- `on_location`: respond to an uploaded location from the user
- `on_subscribe`: respond to a subscribe request
- `on_unsubscribe`: respond to an unsubscribe request
- `on_click`: respond to a user click on a menu
- `on_scan`: respond to a scanned QR code
- `on_location_update`: respond to a location update event
- `on_view`: respond to a user click requesting to load a webpage
- `on_scancode_push`
- `on_scancode_waitmsg`
- `on_pic_sysphoto`
- `on_pic_photo_or_album`
- `on_pic_weixin`
- `on_location_select`

`on_xxx` handlers are defined as follows:

	def on_xxx(self, req):
		return WxResponse()

Handlers take a `WxRequest` parameter `req` and return a `WxResponse` object.

#### WxRequest

A user request is represented by a `WxRequest` object `req`. `req` properties
correspond one-to-one with XML properties, the properties below are common to
all requests:

- ToUserName
- FromUserName
- CreateTime
- MsgType
- MsgId

Depending on the request type, each `req` may have its own set of properties,
but this will always match those defined in the XML. For example, a `MsgType`
for a `text` `req` will have a `Content` variable, while with a `image` `req`
there will be `PicUrl` and `MediaId` instead. For more information, read the
[official documentation](http://mp.weixin.qq.com/wiki/10/79502792eef98d6e0c6e1739da387346.html).

#### WxResponse

`on_xxx` handlers must return a `WxResponse` object. This may be one of the following:

##### WxTextResponse, text messages

 	WxTextResponse("hello", req)
	
##### WxImageResponse, picture messages

	WxImageResponse(WxImage(MediaId='xxyy'),req)
	
##### WxVoiceResponse, voice messages

	WxVoiceResponse(WxVoice(MediaId='xxyy'),req)
	
##### WxVideoResponse, video messages

	WxVideoResponse(WxVideo(MediaId='xxyy', Title='video', Description='test'),req)
	
##### WxMusicResponse, music message

	WxMusicResponse(WxMusic(Title='hey jude', 
		Description='dont make it bad', 
		PicUrl='http://heyjude.com/logo.png', 
		Url='http://heyjude.com/mucis.mp3'), req)

##### WxNewsResponse, (news) article message

	WxNewsResponse(WxArticle(Title='test news', 
		Description='this is a test', 
		Picurl='http://smpic.com/pic.jpg', 
		Url='http://github.com/jeffkit'), req)

##### WxEmptyResponse, empty response
	
	WxEmptyResponse(req)


### Using WxApplication with Django

This is how to implement a `view` response in Django using the `WxApp` above:
	
	from django.http import HttpResponse

	def wechat(request):
		app = WxApp()
		result = app.process(request.GET, request.body)
		return HttpResponse(result)

and in `urls.py`:
	
	urlpatterns = patterns('',
    	url(r'^wechat/', 'myapp.views.wechat'),
	)


### Using WxApplication with Flask

	from flask import request
	from flask import Flask
	app = Flask(__name__)
	
	@app.route('/wechat')
	def wechat():
		app = WxApp()
		return app.process(request.args, request.data)


OK. That's it, `WxApplication` itself has no relevance with the web framework
so you can use whatever you prefer.

### What? You don't like to write WxApplication objects?!

Well, OK, you can write `on_xxx` handlers anywhere, as long as you inform the
`WxApplication` where the response handler is before it is used. For example,
in Django:

	# Write your handler anywhere, I don't care.
	# @any_decorator   # add any decorator you want.
	def my_text_handler(req):
		return WxTextResponse(req.Content, req)
	
	# In the web application:
	def wechat_view(request):
		app = WxApplication()   # 实例化基类就好.
		app.handlers = {'text': my_text_handler} # your preferred interface
		result = app.process(request.GET, request.body, 
			token='xxxx', app_id='xxxx', aes_key='xxxx')
		return HttpResponse(result)
	
		
Yup, define your own message handlers. If you want your own event handlers, I
would tell you to edit `app.event_handlers`, use the same data structures. It
is possible to see the source codes with the correct key. Heh heh.


## 3. OAuth API

OAuth API currently only supports the following interfaces:

- sending messages
- account management
- custom menu administration
- multimedia transfer
- QR codes

Other interfaces may be supported in future. Feel free to add your own.
