# web页面监听消息，登录回调场景

@(开发)[web]

## 主窗口实现
```javascript
<html>
  <head>
  </head>
  <body>
    <p id="showContent"></p>
    <button id="login-btn">Click to Login</button>
    <script>
      /* 此变量保存打开的登录用子窗口 */
      var authWindow;

      /* 此变量保存是否已经登录的状态 */
      var logined = false;

      /* 通过收到子窗口发来的消息而得知登录结果，另一种可能是没有收到结果但是子窗口被关闭，这种情况请查看下面的“目前已知的唯一最佳感知子窗口是否已经被关闭的方式” */
      window.addEventListener('message', function(event) {
        console.log('index.html on message!', event);
        if (event.data.msg === 'auth-result') {
          var htmlText = JSON.stringify(event.data);
          if (event.data.status === 'ok') {
            // toid 从 event.data.toid 取到
            htmlText += "<br><br><span style=\"font-size:20px;\">TOID=" + event.data.toid + "</span>";
            // 注意在此设置登录状态变量
            logined = true;
          }
          setContent(htmlText);
          authWindow.close();
        }
      });

      function startLogin() {
        //var authUrl = `https://open.work.weixin.qq.com/wwopen/sso/qrConnect?appid=wxab249edd27d57738&agentid=1000349&redirect_uri=https://work.medialab.qq.com/&state=eyJhdXRoX3R5cGUiOiJ3ZWJ3ZXdvcmsiLCJhdXRoX2luc3RpZCI6IjEiLCJzZGthcHBpZCI6IjE0MDAwOTgwMTYifQ==`;
        //var authUrl = `./authcall.html`;
        var authUrl = './callback.html';

        /* 打开一个子窗口 */
        authWindow = window.open(authUrl, 'authWindow', "dialog,alwaysRaised,dependent,width=600,height=600,left=600,top=100");

        /* 目前已知的唯一最佳感知子窗口是否已经被关闭的方式 */
        var timer = setInterval(function() {
          if(authWindow.closed) {
            clearInterval(timer);
            /* 如果子窗口已经关闭了，这可能是因为用户自己关闭，也可能是因为登录成功后由本脚本关闭，如何区分？
             * 就是通过检查 logined 变量，如果是成功登录的情况，它的值应该是为 true ；否则，那就是用户自己
             * 关闭了登录窗口，那么我们应该把页面的状态从“等待登录结果”调整为未登录状态 */
            setTimeout(() => {
              if (logined === false) {
                cancelAuth();
              }
            }, 50);
          }
        }, 100);

        // 目前页面的状态是“等待登录结果”
        setContent("Waiting for auth result...");
      }

      // 如果扫码页被关闭，则取消正在登录的状态
      function cancelAuth() {
        setContent("");
        document.getElementById('myModal').style.display = "none";
      }

      function setContent(text) {
        document.getElementById('showContent').innerHTML = text;
      }

      window.onload = function (e) {
        var btn = document.getElementById("login-btn");
        btn.onclick = startLogin;
		//startLogin();
        document.getElementById('closeBtn').onclick = function(e) {
          cancelAuth();
        }
      }
    </script>
  </body>
</html>

```
## 子窗口回调通知
```javascript
<html>
  <body>
    <!-- <a id="sendMsg" href="javascript:void(0)">send message &amp; close</a> -->
    <script>
	  /*
      window.addEventListener('message', function(event) {
        console.log('callback.html on message!', event);
        // IMPORTANT: Check the origin of the data!
        if (~event.origin.indexOf('http://yoursite.com')) { 
            // The data has been sent from your site
            // todo
        } else { 
            // The data hasn't been sent from your site! 
            // Be careful! Do not use it.
            return;
        }
      });
	  */
      window.onload = function(e) {
        var call_back = function () {
          window.opener.postMessage({
            msg: 'auth-result',
            status: 'ok',
            info: '(none)',
            toid: 'jasonchen'
          }, '*');
        }
        // call_back();
      }
    </script>
  </body>
</html>
```