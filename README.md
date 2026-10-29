## Consul-Service-Discovery

#### Complete koro---> [Docker Installation on Ubuntu](https://github.com/Omarmdwasimuddin/Docker-Installation-on-Ubuntu)

####
```bash
ip addr show docker0
```
---

####
```bash
sudo docker run -d --name consule-server -p 8500:8500 --add-host=host.docker.internal:172.17.0.1 consul:1.15
```
---

#### browser e hit koro
```bash
localhost:8500
```
<img width="1542" height="849" alt="image" src="https://github.com/user-attachments/assets/c65d87b3-985c-42cd-a113-2ed7bf304920" />

> consule-server stop hoye gele start korar jonno
```bash
sudo docker start consule-server
```
---

#### ager project open koro
```bash
cd wasuTech-gRPC
```
```bash
code .
```
---

####
```bash
npx nx g @nx/nest:app payment-service
```
---

####
```bash
npx nx g @nx/nest:app checkout-service
```
---

####
```bash
npm i consul
```
---

#### `payment-service/src/main.ts` 
```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app/app.module';
import Consul from 'consul';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const port = 3005;

  const consul = new Consul({ 
    host: 'localhost',
    port: 8500,
   })
   
   const serviceId = 'payment-unique-id-1';
   const registrationDetails = {
    name: 'payment-service',
    address: 'host.docker.internal',
    port: port,
    id: serviceId,
    check: {
      name: 'payment-service-health',
      http: `http://host.docker.internal:${port}/api/health`,
      interval: '10s',
      timeout: '5s',
    }
   }

    await consul.agent.service.register(registrationDetails);
   process.on('SIGINT', async () => {
    await consul.agent.service.deregister(serviceId);
    process.exit();
   })

   await app.listen(port);
   console.log(`Payment service is running on port ${port} and registered in Consul with ID: ${serviceId}`);
}

bootstrap();
```
---

#### `payment-service/src/app/app.controller.ts`
```ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller('api')
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('health')
  checkHealth(){
    return { status: 'UP' }
  }
}
```
---

#### `checkout-service/src/app/app.service.ts`
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
    const paymentInfo = services['payment-unique-id-1']

    if(!paymentInfo) {
      throw new Error('Payment Service not found in registry!')
    }

    const address = 'localhost';
    const port = paymentInfo.Port;
    const finalUrl = `http://${address}:${port}/api/health`
    console.log(`Discovery Succussfull calling ${finalUrl}`)

    return {
      message: 'Service Discovered Successfully!',
      discoverUrl: finalUrl,
      serviceData: paymentInfo
    }
  }
}
```
---

#### `checkout-service/src/app/app.controller.ts`
```ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller('checkout')
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('process')
  async processCheckout() {
    return this.appService.discoverAndCallPayment();
  }
}
```
---


####
```bash
npx nx serve payment-service
```
<img width="1358" height="852" alt="image" src="https://github.com/user-attachments/assets/ee7aba5b-8a2b-4757-9b8f-061e8c86ee70" />

> amader payment-service ta ke run korte wsl diye run korte hobe jate error na ashe process niche deya
> powershall open koro
```bash
wsl -d Ubuntu
```
> WSL-e Node ache kina check
```bash
node -v && npm -v
```
> na thakle install korte hobe
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```
```bash
mkdir -p ~/projects
rsync -a --exclude node_modules --exclude dist --exclude .nx /mnt/c/Users/User/Desktop/wasuTech-gRPC/ ~/projects/wasuTech-gRPC/
cd ~/projects/wasuTech-gRPC
npm install
npx nx serve payment-service
```
> arekta WSL Terminal e daw
```bash
curl http://localhost:3005/api/health
```
<img width="1064" height="344" alt="image" src="https://github.com/user-attachments/assets/473b5966-0eed-4a44-83a8-b97967d47e69" />

---

####
```bash
npx nx serve checkout-service
```
> browser e hit koro
```bash
localhost:3000/api/checkout/process
```
<img width="467" height="402" alt="image" src="https://github.com/user-attachments/assets/7293b38e-f12f-429e-bb16-e846e0fcfcd5" />


---
