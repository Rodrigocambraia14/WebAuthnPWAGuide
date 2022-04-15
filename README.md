# WebAuthnPWAGuide
Biometry Authentication Guide with WebAuthn in a PWA w/ Ionic and Angular

_A credencial WebAuthn é basicamente uma key atrelada ao dispositivo do usuário, portanto, um dispositivo pode possui apenas um acesso biométrico por vez para o mesmo App._


:grey_exclamation: - **Início**

  A primeira coisa que devemos fazer, é verificar se o dispositivo do usuário que está acessando a aplicação tem suporte para autenticação por biometria / senha de acesso do dispositivo.


```
window.PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable()
    .then((available) => {
      if (available)
         this.biometricAuthAvailable = true;
    })
```

_**this.biometricAuthAvailable** é iniciado pelo componente com o valor **false**, e só se torna **true** caso o dispositivo possua suporte à esse tipo de acesso._


:grey_exclamation: - **Criando o objeto necessário para solicitar as credenciais**

`credentialOptions: PublicKeyCredentialCreationOptions;`

```
this.credentialOptions = {
              challenge: new Uint8Array([
                // must be a cryptographically random number sent from a server
                0x8c, 0x0a, 0x26, 0xff, 0x22, 0x91, 0xc1, 0xe9, 0xb9, 0x4e, 0x2e, 0x17,
                0x1a, 0x98, 0x6a, 0x73, 0x71, 0x9d, 0x43, 0x48, 0xd5, 0xa7, 0x6a, 0x15,
                0x7e, 0x38, 0x94, 0x52, 0x77, 0x97, 0x0f, 0xef,
              ]).buffer,
              rp: {
                id: `${window.location.hostname}`, 
                name: 'ClubApp',
              },
              authenticatorSelection: {
                authenticatorAttachment: "platform",
                userVerification: "discouraged"
              },
              user: {
                id: new Uint8Array(16),
                name: `${this.formControls.email.value}`,
                displayName: 'Usuario ClubApp',
              },
              pubKeyCredParams: [{ alg: -7, type: 'public-key' }],
            };
```

_**this.credentialOptions** é um objeto que precisa ser montado antes de solicitarmos a criação da credencial biométrica no dispositivo do usuário, para isso precisamos enviar uma:_

- **challenge** 
       - _Nada mais do que um array Buffer randômico_.

- **um rp (relying party)** 
       - _Deve ser equivalente ao aplicativo que está requisitando o acesso, sendo o **ID = domínio do seu PWA / APP, NAME = nome do seu projeto / aplicação**._ 

- **user** 
       - _Por fim, informações sobre o usuário, passando algum identificador em name, e em display name, apenas atrelar ele ao projeto com uma nomenclatura fixa._


:grey_exclamation: - **Solicitando credenciais**


```
const pkCredCreateOpts = this.credentialOptions;
        
              pkCredCreateOpts.user.id = pkCredCreateOpts.user.id // convert from base64 to ab
              const challengeStr = pkCredCreateOpts.challenge // keep for later - base64url
              pkCredCreateOpts.challenge = pkCredCreateOpts.challenge
        
              this.credential = await navigator.credentials.create({
                publicKey: pkCredCreateOpts 
              })
```

_Ao acessar **credentials.create**, o aplicativo irá solicitar o acesso do dispositivo, seja por biometria ou senha padrão de desbloqueio de tela, e caso a resposta seja bem sucedida, retornará novas informações que serão atribuídas à **credentials.response**_

:grey_exclamation: - **Decodificando informações**

_Agora precisamos tratar os dados que recebemos através do **credentials.create**, a fim de conseguirmos utilizá-los posteriormente no lado do servidor, e salvarmos um **id** atrelado ao usuário._

 
```
const utf8Decoder = new TextDecoder('utf-8')
              const decodedClientData = utf8Decoder.decode(this.credential.response.clientDataJSON)
              // // parse the string as an object
              const clientDataObj = JSON.parse(decodedClientData);
        
              // note: a CBOR decoder library is needed here.
        
              const cbor = require('cbor-web')
        
              let encoded = cbor.encode(true) // Returns <Buffer f5>
              
               const decodedAttestationObj = cbor.decode(this.credential.response.attestationObject)

const {authData} = decodedAttestationObj;
        
              // // get the length of the credential ID
              const dataView = new DataView(new ArrayBuffer(2));
              const idLenBytes = authData.slice(53, 55);
              idLenBytes.forEach((value, index) => dataView.setUint8(index, value));
              const credentialIdLength = dataView.getUint16(0);
        
              // // get the credential ID
               const credentialId = authData.slice(55, 55 + credentialIdLength);
        
              // // get the public key object
               const publicKeyBytes = authData.slice(55 + credentialIdLength);
               
              // // the publicKeyBytes are encoded again as CBOR
               const publicKeyObject = cbor.decode(publicKeyBytes.buffer);
        
               credentialId.map(x => this.credentialCode.push(x));
```

_Para o trecho acima, é necessário utilizarmos uma lib de CBOR encode / decode, sugiro **npm i cbor-web**, e então conseguimos acessar suas propriedades através do **const cbor = require('cbor-web')**_

_Basicamente, todo o trecho acima são sequências de conversões e decodificações atreladas à arrayBuffers, Jsons e CBOR, para que seja possível transformar a credencial de biometria em algo possível de salvar em um banco de dados ou um localStorage_


:grey_exclamation: - **Montando seu objeto de request**

_Agora queremos salvar esse Id na nossa base de dados, e atrelar ela ao usuário que será cadastrado em nosso sistema_


```
const request = {
                 biometryId: this.credentialCode.toString(),
                 email: this.formControls.email.value,
                 cpf: this.formControls.username.value.replace(/\D+/g, '')
               }
        
               this.biometryService.create(request).subscribe((res) => {
                if (res.valid) 
                {
                  localStorage.setItem('BiometryAccess', 'enabled');
                  localStorage.setItem('BiometryId', JSON.stringify(credentialId));
                  this.toastService.presentSuccessToast(res.message);
                  this.router.navigateByUrl('/auth/login');
                }
                else
                  this.toastService.presentErrorToast(res.data.erros);
               })
```

_No código acima temos **biometryId (uma sequência de números no formato string = '[13, 53 ,256 , 1 ,56, 65 ,66 ...]')** que é o que nos interessa nesse tópico. Recomendo salvá-lo em seu servidor como um **nvarchar**, assim como recomendo **criptografar** essa informação antes de acoplá-la na sua base de dados._

_Após salvar o usuário e sua biometria no servidor, e obter o retorno de sucesso, é necessário armazenarmos essas informações em **localStorage**, para isso, recomendo um **boolean** que diga se o acesso biométrico foi habilitado pelo usuário ou não, e a própria **credencial** em si._ 

:exclamation::exclamation:  **OBS -** _Ao salvar a credencial no localStorage, **NÃO** podemos esquecer de utilizar o **.JsonStringify(id);** como foi feito no exemplo acima._ :exclamation::exclamation:

:star: **Agora temos a etapa de cadastro concluída** :star:


:grey_exclamation: - **Autenticando o usuário com biometria**

_Verificamos se o acesso Biométrico já foi ativado anteriormente naquele dispositivo, e caso o retorno seja true, chamamos o método que trata o login por biometria._


```
if (localStorage.getItem('BiometryAccess') == 'enabled')
      return this.handleBiometrics();
```

_:exclamation::exclamation: **OBS -** é importante que o método handleBiometrics seja **ASSÍNCRONO**, para isso, basta declará-lo como async handleBiometrics() { }_:exclamation::exclamation:


 _Dentro do método citado acima, buscaremos o Id armazenado em localStorage através do código abaixo, com os devidos tratamentos de dados para obtê-lo no formato desejado._

_Segue o código modelo:_
```
let storedCredential = localStorage.getItem('BiometryId');

    const credential = JSON.parse(storedCredential);

    const credentialId = new Uint8Array(credential.data);
```

:grey_exclamation:- **Criando as credentialOptions para a autenticação**

_Novamente, precisamos criar o objeto necessário para requisitar o acesso por biometria no dispositivo do usuário._


```
const publicKeyCredentialRequestOptions = {
           challenge: new Uint8Array([
             0x8c, 0x0a, 0x26, 0xff, 0x22, 0x91, 0xc1, 0xe9, 0xb9, 0x4e, 0x2e, 0x17,
             0x1a, 0x98, 0x6a, 0x73, 0x71, 0x9d, 0x43, 0x48, 0xd5, 0xa7, 0x6a, 0x15,
             0x7e, 0x38, 0x94, 0x52, 0x77, 0x97, 0x0f, 0xef,
           ]).buffer,
           allowCredentials: [{
             id: credentialId,
             type: 'public-key' as any,
             transports: ['usb', 'ble', 'nfc'] as any,
         }],
           userVerification: 'discouraged' as any,
           timeout: 60000,
       }
```

_:exclamation::exclamation:_Os casts **'as any'** são necessários para que não tenhamos problemas com a tipagem do typescript, que entra em conflito com os tipos fornecidos pelo navigator.credentials__:exclamation::exclamation:

- **Solicitando o acesso por biometria**


```
const assertion : any = await navigator.credentials.get({
           publicKey: publicKeyCredentialRequestOptions
       });
       
       var request = assertion.rawId;

       request = new Uint8Array(request);

       this.store.dispatch(new LoginBiometry({ biometryId: request.toString()}));
```

_`No trecho acima, obtivemos as informações provindas da tentativa de acesso por biometria do usuário, e utilizaremos o request.toString() para enviá-lo ao servidor, criptografar e comparar com a informação armazenada no nosso banco de dados, caso consigamos encontrar o usuário através do Id biométrico, podemos logá-lo com sucesso.`_

:exclamation::exclamation: _Caso queria uma verificação adicional antes de logar seu usuário, você pode salvar o dispositivo utilizado no momento de cadastro em sua base de dados utilizando o comando navigator.userAgent, e no momento de login, buscá-lo e compará-lo junto com o Id fornecido pela biometria, garantindo mais um nível de segurança na sua autenticação._:exclamation::exclamation:

:star: **Agora temos a etapa de login concluída** :star:


_Até que um dia..._

_Você se depara com um usuário que trocou de aparelho, porém não quer utilizar o login convencional e está exigindo sua biometria de volta, ou até mesmo alguém que já possui cadastro, mas quer utilizar o serviço de biometria ! O que fazer, caro dev?_ :cold_sweat:

Simples! :blush:

:grey_exclamation:- **Atualizando credenciais biométricas**

_Após logado, você pode disponibilizar uma tela, modal, ou a ferramenta que julgar conveniente, para permitir que o usuário atualize, altere, ou remova seu acesso biométrico._

_Mas como_ :question:


```
const jsonbiometryId = JSON.parse(localStorage.getItem('BiometryId'))

    const arrayBiometryId = new Uint8Array(jsonbiometryId.data);

    const biometryId = arrayBiometryId.toString();

    if (biometryId != undefined && biometryId != null)
    {
      const request = {
        userId : this.userId,
        biometryId : biometryId
      }

      this.biometryService.check(request).subscribe((res) => {
        if (res.valid)
        {
         this.isCurrentUserBiometry = res.data;
          if (this.isCurrentUserBiometry)
          {
            this.biometryText = 'Atualizar Biometria'
            this.alreadyHasBiometry = true;
          }
          else
          {
            this.biometryText = 'Cadastrar Biometria'
            this.alreadyHasBiometry = false;
          }
        }
      })
      
    }
    else
    {
      this.biometryText = 'Cadastrar Biometria'
      this.alreadyHasBiometry = false;
    }
```

_Assim que ocomponente é inicializado, buscamos por uma credencial armazenada no localStorage, e caso ela não seja nula ou indefinida, enviamos a mesma para o servidor, junto com o Id do usuario logado, e verificamos se esse Id biométrico é de fato pertencente àquele usuário autenticado na aplicação. Caso a resposta seja sim, habilitamos o layout de atualização da biometria, caso seja não, habilitamos o cadastro de biometria, visto que o usuário logado não possui nenhuma credencial vinculada à ele._

**Mas, por que é necessário verificar a credencial, se ele já está logado :question::question:**

_Sabendo que em um celular, outra pessoa pode acessar outra conta no mesmo aplicativo, a biometria armazenada no localStorage não necessariamente pertence àquele usuário, por isso a importância de armazenar a credencial da biometria no banco de dados e atrelar ela ao usuário cadastrado, para fins de comparação e checagem_

:exclamation::exclamation: _Caso haja **biometria no localStorage**, porém ela **não** pertença ao usuário, caso este queira prosseguir com a criação de acesso biométrico para a conta do mesmo, torna-se necessário avisá-lo de que a biometria antiga cadastrada no mesmo dispositivo para outra conta, será **substituída** pela biometria atual cadastrada por ele._:exclamation::exclamation: 

- **Voltando ao cadastro / atualização de biometria**

Após sabermos se a necessidade do usuário é cadastrar ou atualizar a biometria, realizamos o mesmo procedimento utilizado para criar a credencial no inicio da nossa wiki.


```
const pkCredCreateOpts = this.credentialOptions;

      pkCredCreateOpts.user.id = pkCredCreateOpts.user.id // convert from base64 to ab
      const challengeStr = pkCredCreateOpts.challenge // keep for later - base64url
      pkCredCreateOpts.challenge = pkCredCreateOpts.challenge

      this.credential = await navigator.credentials.create({
        publicKey: pkCredCreateOpts 
      }) 

      // // decode the clientDataJSON into a utf-8 string
      const utf8Decoder = new TextDecoder('utf-8')
      const decodedClientData = utf8Decoder.decode(this.credential.response.clientDataJSON)
      // // parse the string as an object
      const clientDataObj = JSON.parse(decodedClientData);

      // note: a CBOR decoder library is needed here.

      const cbor = require('cbor-web')

      let encoded = cbor.encode(true) // Returns <Buffer f5>
      
       const decodedAttestationObj = cbor.decode(this.credential.response.attestationObject)
  
      const {authData} = decodedAttestationObj;

      // // get the length of the credential ID
      const dataView = new DataView(new ArrayBuffer(2));
      const idLenBytes = authData.slice(53, 55);
      idLenBytes.forEach((value, index) => dataView.setUint8(index, value));
      const credentialIdLength = dataView.getUint16(0);

      // // get the credential ID
       const credentialId = authData.slice(55, 55 + credentialIdLength);

      // // get the public key object
       const publicKeyBytes = authData.slice(55 + credentialIdLength);
       
      // // the publicKeyBytes are encoded again as CBOR
       const publicKeyObject = cbor.decode(publicKeyBytes.buffer);

       credentialId.map(x => this.credentialCode.push(x));

       const request = {
         biometryId: this.credentialCode.toString(),
         userId: this.userId,
       }
```

_Após montarmos o objeto de request, chamamos o endpoint desejado para criar / atualizar as credenciais do usuário._

_Na sua API, você pode verificar se o usuário já possui credencial, caso tenha, atualize, caso contrário, crie._


_Logo após recebermos a resposta do servidor na API, armazenamos a nova credencial no localStorage e disparamos uma mensagem de sucesso ao usuário._


```
localStorage.setItem('BiometryAccess', 'enabled');
          localStorage.setItem('BiometryId', JSON.stringify(credentialId));
          this.toastService.presentSuccessToast(res.message);
```

- **Deletando uma credencial**

_Caso julgue necessário oferecer essa opção, basta criar uma API que busque pelo Id do usuário que está realizando o request, remover ou inativar a credencial da sua base de dados, e logo em seguida removê-la do localStorage._

:star::star: **Hints** :star::star:

_Sempre englobar seu método que busca / cadastra credenciais com um try catch, e oferecer uma via de saída em caso de erro (direcionar para login convencional, por exemplo)_

_Para realizar testes de biometria em localhost, o relyingParty Id deve ser 'localhost'_

**Configuração pro dev tools**

_Clique nos 3 pontos ao lado da engrenagem -_

![image](https://user-images.githubusercontent.com/77635828/163494902-e23890ca-1578-4612-9d76-e0141b79f407.png)

_Busque a opção **'more tools'** -_

![image](https://user-images.githubusercontent.com/77635828/163494915-6d5950fe-91d9-46a8-8960-8e7e2a5caa9b.png)

_Procure e clique na opção **WebAuthn**_

_Marque a opção exibida na imagem_

![image](https://user-images.githubusercontent.com/77635828/163494934-e06befb7-2abe-4bb3-9bd9-d2a0d73f8833.png)



_Siga a configuração abaixo_

![image](https://user-images.githubusercontent.com/77635828/163495089-2859a88b-28ca-443f-91a1-c41c760a2347.png)

_Clique em **'Add'**_

_E pronto, agora é só ser feliz !_


:star:  **Fim**  :star:

_`made by: Rodrigo Cambraia`_

