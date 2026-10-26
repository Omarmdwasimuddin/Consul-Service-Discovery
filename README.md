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
