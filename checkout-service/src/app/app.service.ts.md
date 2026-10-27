# NestJS-এ Consul Service Discovery — লাইন বাই লাইন ব্যাখ্যা

> `AppService` কীভাবে Consul থেকে রেজিস্টার করা সার্ভিসের তথ্য নিয়ে `payment-service` খুঁজে বের করে এবং Health Check URL তৈরি করে — সহজ বাংলায়।

> ⚠️ **গুরুত্বপূর্ণ:** বর্তমান কোডটি Payment Service-এর **URL তৈরি করছে**, কিন্তু সেই URL-এ আসলে **HTTP request পাঠাচ্ছে না**।

---

## 📑 সূচিপত্র

1. [import স্টেটমেন্ট](#১-import-স্টেটমেন্ট)
2. [@Injectable() ও AppService](#২-injectable-এবং-appservice)
3. [Consul-এর সঙ্গে সংযোগ](#৩-consul-এর-সঙ্গে-সংযোগ)
4. [discoverAndCallPayment()](#৪-discoverandcallpayment-method)
5. [সার্ভিসের তালিকা নেওয়া](#৫-consul-থেকে-সার্ভিসের-তালিকা-নেওয়া)
6. [Payment Service খুঁজে বের করা](#৬-নির্দিষ্ট-payment-service-খুঁজে-বের-করা)
7. [পাওয়া গেছে কি না পরীক্ষা](#৭-payment-service-পাওয়া-গেছে-কি-না-পরীক্ষা)
8. [address ও port](#৮-সার্ভিসের-address-এবং-port-নেওয়া)
9. [finalUrl তৈরি](#৯-finalurl-তৈরি-করা)
10. [console.log](#১০-console-এ-message-দেখানো)
11. [Response return](#১১-response-return-করা)
12. [কোডে কী উন্নতি দরকার](#-কোডে-কী-উন্নতি-দরকার)
13. [Registration vs Discovery vs Communication](#-তিনটি-ধাপের-পার্থক্য)
14. [সম্পূর্ণ কোড](#-সম্পূর্ণ-কোড-একসঙ্গে)
15. [Checklist](#-checklist)

---

## ১. `import` স্টেটমেন্ট

```ts
import { Injectable } from "@nestjs/common";
import Consul from "consul";
```

| Import | ব্যাখ্যা |
| --- | --- |
| `Injectable` | NestJS-এর decorator। class-কে **Dependency Injection (DI)** system-এ ব্যবহারযোগ্য করে, ফলে `AppService`-কে অন্য class-এ inject করা যায় |
| `Consul` | `consul` npm package-এর class। এর মাধ্যমে Consul Agent-এর API-তে request পাঠানো যায় (সার্ভিসের তালিকা নেওয়া, সার্ভিস খোঁজা, তথ্য সংগ্রহ করা) |

---

## ২. `@Injectable()` এবং `AppService`

```ts
@Injectable()
export class AppService {
```

| অংশ | কাজ |
| --- | --- |
| `@Injectable()` | NestJS-কে জানায়, এই class DI-এর মাধ্যমে সরবরাহ করা যাবে |
| `export` | অন্য ফাইল থেকে class-টি import করার সুযোগ দেয় |
| `class AppService` | business logic রাখার class |

> 💡 সহজভাবে: **Controller** request গ্রহণ করে, আর **Service** সেই request-এর প্রয়োজনীয় কাজ করে।

---

## ৩. Consul-এর সঙ্গে সংযোগ

```ts
private consul = new Consul({
  host: 'localhost',
  port: 8500,
})
```

`consul` নামে একটি property তৈরি করে Consul client initialize করা হচ্ছে।

| কোড | কাজ |
| --- | --- |
| `private` | property-টি class-এর বাইরে সরাসরি ব্যবহার করা যায় না |
| `consul` | Consul client রাখার property-র নাম |
| `new Consul(...)` | নতুন Consul client তৈরি করে |
| `host: 'localhost'` | Consul কোথায় চলছে |
| `port: 8500` | Consul-এর HTTP API port |

**উদাহরণ:** Consul যদি Docker-এ `8500` পোর্টে চালু থাকে, client সাধারণত `localhost:8500`-এ যোগাযোগ করবে।

> ⚠️ NestJS নিজেও Docker container-এর ভেতরে চললে `localhost` বলতে NestJS container-টিকেই বোঝাতে পারে। সে ক্ষেত্রে সঠিক Consul host ব্যবহার করতে হবে।

---

## ৪. `discoverAndCallPayment()` method

```ts
async discoverAndCallPayment() {
```

এটি একটি asynchronous method। নামটি ভেঙে বুঝি:

| অংশ | অর্থ |
| --- | --- |
| `discover` | সার্ভিস খুঁজে বের করা |
| `callPayment` | Payment Service-কে call করার উদ্দেশ্য |
| `async` | ভেতরে `await` ব্যবহার করা যায় |

**মূল কাজ:** Consul থেকে সার্ভিসের তথ্য সংগ্রহ করে Payment Service-এর URL তৈরি করা।

---

## ৫. Consul থেকে সার্ভিসের তালিকা নেওয়া

```ts
const services = await this.consul.agent.service.list();
```

কোডের অন্যতম গুরুত্বপূর্ণ লাইন।

| অংশ | অর্থ |
| --- | --- |
| `this.consul` | আগে তৈরি করা Consul client |
| `.agent.service.list()` | Consul Agent-এ নিবন্ধিত সার্ভিসগুলোর তালিকা চায় |
| `await` | Consul-এর response আসা পর্যন্ত অপেক্ষা করে |
| `const services` | পাওয়া তথ্য রাখার ভেরিয়েবল |

**উদাহরণ:** Consul-এ এই সার্ভিস নিবন্ধিত থাকলে:

```
ID:      payment-unique-id-1
Name:    payment-service
Address: host.docker.internal
Port:    3005
```

`services`-এর মধ্যে এই সার্ভিসের তথ্য পাওয়া যেতে পারে। সঠিক response নির্ভর করে `consul` package-এর API এবং registry-র তথ্যের ওপর।

---

## ৬. নির্দিষ্ট Payment Service খুঁজে বের করা

```ts
const paymentInfo = services['payment-unique-id-1'];
```

`services` object থেকে নির্দিষ্ট **ID** দিয়ে Payment Service-এর তথ্য বের করা হচ্ছে।

ধরো `services`-এ এমন entry আছে:

```js
{
  'payment-unique-id-1': {
    ID: 'payment-unique-id-1',
    Service: 'payment-service',
    Address: 'host.docker.internal',
    Port: 3005
  }
}
```

তাহলে এই entry-টি `paymentInfo`-তে জমা হবে।

**কেন ID ব্যবহার করা হয়েছে?**
একই নামে একাধিক instance থাকতে পারে। প্রতিটির ID আলাদা হলে নির্দিষ্ট instance শনাক্ত করা সহজ।

> ⚠️ `paymentInfo`-এর field-এর নাম ও structure তোমার ব্যবহৃত `consul` package-এর actual response-এর সঙ্গে মিলিয়ে নিতে হবে।

---

## ৭. Payment Service পাওয়া গেছে কি না পরীক্ষা

```ts
if (!paymentInfo) {
  throw new Error('Payment Service not found in registry!')
}
```

| অংশ | অর্থ |
| --- | --- |
| `!paymentInfo` | `paymentInfo` না থাকলে condition সত্য |
| `throw new Error(...)` | error তৈরি করে method-এর স্বাভাবিক execution বন্ধ করে |
| message | Consul registry-তে Payment Service পাওয়া যায়নি |

**Error message:**

```
Payment Service not found in registry!
```

সার্ভিস পাওয়া গেলে পরের লাইনগুলো execute হবে।

---

## ৮. সার্ভিসের address এবং port নেওয়া

```ts
const address = 'localhost';
const port = paymentInfo.port;
```

### 🔹 প্রথম লাইন — `address`

Payment Service-এর address হিসেবে `localhost` **hardcode** করা হয়েছে।

> ⚠️ **সমস্যা:** Consul-এ register করার সময় `host.docker.internal` দেওয়া হয়েছিল, অথচ এখানে `localhost` ব্যবহার হচ্ছে।
> Service Discovery-র উদ্দেশ্য হলো registry থেকে সার্ভিসের **প্রকৃত address ও port** নেওয়া। তাই address hardcode না করে Consul-এর response থেকে নেওয়াই উপযুক্ত।

### 🔹 দ্বিতীয় লাইন — `port`

`paymentInfo` থেকে port নেওয়ার চেষ্টা।

> ⚠️ `consul.agent.service.list()` response-এ সাধারণত metadata-তে **`Port`** (বড় হাতের `P`) থাকে। তোমার package-এর actual response-এ `port` নাকি `Port` আছে, যাচাই করো। ভুল property দিলে `undefined` আসতে পারে।

---

## ৯. `finalUrl` তৈরি করা

```ts
const finalUrl = `http://${address}:${port}/api/health`;
```

JavaScript-এর **Template Literal** দিয়ে URL তৈরি হচ্ছে।

ধরো `address = 'localhost'` এবং `port = 3005`, তাহলে URL:

```
http://localhost:3005/api/health
```

| অংশ | অর্থ |
| --- | --- |
| `http://` | HTTP protocol |
| `${address}` | সার্ভিসের address |
| `${port}` | সার্ভিসের port |
| `/api/health` | Payment Service-এর Health Check endpoint |

এই URL-এ request পাঠালে সার্ভিস চালু থাকলে এবং endpoint কাজ করলে `{"status":"UP"}` পাওয়ার কথা।

---

## ১০. Console-এ message দেখানো

```ts
console.log(`Discovery Succussfull calling ${finalUrl}`);
```

Terminal-এ message দেখায়:

```
Discovery Succussfull calling http://localhost:3005/api/health
```

> ⚠️ message-এ `calling` লেখা থাকলেও এই লাইন **শুধু log করে**, নিজে কোনো HTTP request পাঠায় না।
>
> 📝 ছোট একটি বানান-টিপ: `Succussfull` হবে `Successful`।

---

## ১১. Response return করা

```ts
return {
  message: 'Service Discovered Successfully!',
  discoverUrl: finalUrl,
  serviceData: paymentInfo
}
```

| Property | অর্থ |
| --- | --- |
| `message` | Service Discovery সফল হয়েছে বলে message |
| `discoverUrl` | তৈরি করা Payment Service URL |
| `serviceData` | Consul থেকে পাওয়া সার্ভিসের তথ্য |

Controller এই method call করে return value client-কে পাঠালে JSON response এমন হতে পারে:

```json
{
  "message": "Service Discovered Successfully!",
  "discoverUrl": "http://localhost:3005/api/health",
  "serviceData": {
    "ID": "payment-unique-id-1",
    "Service": "payment-service",
    "Port": 3005
  }
}
```

> এটি বোঝানোর উদাহরণ। প্রকৃত `serviceData`-এর structure Consul response অনুযায়ী হবে।

---

## 🛠️ কোডে কী উন্নতি দরকার

বর্তমান কোড Service Discovery করে এবং URL তৈরি করে। কিন্তু Payment Service-কে বাস্তবে call করতে:

1. HTTP request পাঠাতে হবে।
2. Consul response থেকে সঠিক `address` ও `port` নিতে হবে।

### ✅ Consul response (`ID`, `Address`, `Port`) অনুযায়ী উন্নত রূপ

```ts
const services = await this.consul.agent.service.list();
const paymentInfo = services['payment-unique-id-1'];

if (!paymentInfo) {
  throw new Error('Payment Service not found in registry!');
}

const address = paymentInfo.Address;
const port = paymentInfo.Port;
const finalUrl = `http://${address}:${port}/api/health`;

console.log(`Discovered Payment Service: ${finalUrl}`);
```

> ⚠️ এটিও শুধু URL তৈরি করবে। HTTP request পাঠাতে `fetch()` বা `HttpService` ব্যবহার করতে হবে।
> পাশাপাশি Consul থেকে পাওয়া address-টি যে পরিবেশ থেকে request পাঠানো হচ্ছে, সেখান থেকে **reachable** কি না নিশ্চিত করতে হবে।

---

## 🔄 তিনটি ধাপের পার্থক্য

| ধাপ | কাজ |
| --- | --- |
| **১. Service Registration** | সার্ভিসের তথ্য Consul-এ রাখা |
| **২. Service Discovery** | Consul থেকে সার্ভিসের ঠিকানা খুঁজে বের করা |
| **৩. Service-to-Service Communication** | সেই ঠিকানায় HTTP বা অন্য protocol দিয়ে বাস্তবে request পাঠানো |

```mermaid
flowchart LR
    A[Payment Service<br/>Register করে] --> B[(Consul Registry)]
    C[AppService<br/>list() দিয়ে খোঁজে] --> B
    B --> D[Address + Port পায়]
    D --> E[finalUrl তৈরি]
    E -. এখনও বাকি .-> F[HTTP Request পাঠানো]
```

> তোমার কোডটি মূলত **প্রথম দুটি ধাপের মধ্যে Service Discovery অংশটি** বাস্তবায়ন করছে। তৃতীয় ধাপ (সত্যিকারের call) এখনো বাকি।

---

## 🧩 সম্পূর্ণ কোড একসঙ্গে

> উপরের অংশগুলো জুড়ে দেওয়া মূল কোড (তোমার বর্তমান রূপ):

```ts
import { Injectable } from "@nestjs/common";
import Consul from "consul";

@Injectable()
export class AppService {
  private consul = new Consul({
    host: 'localhost',
    port: 8500,
  })

  async discoverAndCallPayment() {
    const services = await this.consul.agent.service.list();

    const paymentInfo = services['payment-unique-id-1'];

    if (!paymentInfo) {
      throw new Error('Payment Service not found in registry!')
    }

    const address = 'localhost';
    const port = paymentInfo.port;

    const finalUrl = `http://${address}:${port}/api/health`;

    console.log(`Discovery Succussfull calling ${finalUrl}`);

    return {
      message: 'Service Discovered Successfully!',
      discoverUrl: finalUrl,
      serviceData: paymentInfo
    }
  }
}
```

---

## ✅ Checklist

- [ ] `address` hardcode না করে `paymentInfo.Address` ব্যবহার করা
- [ ] `port`-এর বদলে `paymentInfo.Port` (আসল property-র নাম যাচাই করে)
- [ ] `fetch()` বা `HttpService` দিয়ে সত্যিকারের HTTP request পাঠানো
- [ ] Consul `host`/`port` environment variable থেকে নেওয়া
- [ ] serviceId hardcode না করে name দিয়ে খোঁজা বা dynamic করা
- [ ] Docker-এ চললে address reachable কি না নিশ্চিত করা
- [ ] log message-এর বানান ঠিক করা (`Successful`)
