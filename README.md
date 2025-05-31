
`laravel/vonage-notification-channel`e :`guzzlehttp/guzzle`

```
composer require laravel/vonage-notification-channel guzzlehttp/guzzle
```
O pacote inclui um [arquivo de configuração](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php) .No entanto, você não precisa exportar esse arquivo para o seu aplicativo. Você pode simplesmente usar as `VONAGE_KEY`variáveis `VONAGE_SECRET`​​de ambiente e para definir suas chaves públicas e secretas do Vonage.

Após definir suas chaves, você deve definir uma `VONAGE_SMS_FROM`variável de ambiente que defina o número de telefone para o qual suas mensagens SMS devem ser enviadas por padrão. Você pode gerar esse número de telefone no painel de controle da Vonage:
```
VONAGE_SMS_FROM=15556666666
```


```
composer require vonage/client
```







# Implementação de Recuperação de Senha por SMS

  

Este documento explica a funcionalidade de recuperação de senha por SMS. Esta funcionalidade permite aos usuários redefinir suas senhas via SMS além do método tradicional baseado em email.

  

## Visão Geral

  

O sistema de recuperação de senha por SMS oferece uma forma alternativa para os usuários recuperarem o acesso às suas contas quando esquecem suas senhas. Em vez de receberem um email com um link de redefinição, os usuários podem optar por receber uma nova senha gerada automaticamente via SMS no número de telefone cadastrado.

  

## Arquivos Modificados

  

1. **Modelo de Usuário** (`app/Models/User.php`)

   - Adicionado método `routeNotificationForVonage` para formatar corretamente os números de telefone para envio de SMS

   - Adicionado tratamento de segurança de tipos e valores nulos para números de telefone

  

2. **Notificação ShortMessageService** (`app/Notifications/ShortMessageService.php`)

   - Criada classe de notificação para lidar com o envio de SMS

   - Implementa segurança de tipos e formatação adequada para mensagens SMS

  

3. **CustomResetPasswordNotification** (`app/Filament/Auth/CustomResetPasswordNotification.php`)

   - Modificada a página de solicitação de redefinição de senha para adicionar a opção de recuperação por SMS

   - Adicionados métodos para lidar com recuperação baseada em telefone

   - Implementa redefinição direta de senha e notificação via SMS

  

4. **Configuração de Serviços** (`config/services.php`)

   - Adicionada seção de configuração do Vonage para envio de SMS

  

## Como Funciona

  

### Interface do Usuário

  

A página de recuperação de senha oferece dois métodos para recuperação:

- **Email** - Método tradicional que envia um link de redefinição

- **SMS** - Método alternativo que envia uma senha nova diretamente

  

O usuário seleciona o método preferido e insere seu endereço de email ou número de telefone.

  

### Processo de Recuperação

  

#### Fluxo de Recuperação por Email:

1. O usuário insere seu endereço de email

2. O sistema envia um email com um link de redefinição de senha

3. O usuário clica no link e define uma nova senha

  

#### Fluxo de Recuperação por SMS:

1. O usuário insere seu número de telefone

2. O sistema valida o número de telefone comparando com os usuários cadastrados

3. O sistema gera uma nova senha segura

4. O sistema atualiza a senha do usuário no banco de dados

5. O sistema envia a nova senha via SMS

6. O usuário faz login com a nova senha

7. O sistema solicita que o usuário altere sua senha no próximo login

  

### Implementação Técnica

  

#### Formatação do Número de Telefone

  

O método `routeNotificationForVonage` no modelo User trata a formatação adequada do número de telefone:

```php

public function routeNotificationForVonage($notification): string

{

    // Garante que temos um número de telefone válido para evitar problemas com nulos

    $phoneNumber = $this->phone ?? '';

    // Formata o número de telefone corretamente para o Vonage (seguro contra nulos)

    $phone = preg_replace('/[^0-9]/', '', $phoneNumber);

    // Adiciona o código do país Brasil (55) se não estiver presente e o número tiver 11 dígitos (DDD + número)

    if (strlen($phone) === 11 && str_starts_with($phone, '55') === false) {

        $phone = '55' . $phone;

    }

    // Sempre retorna uma string, mesmo que vazia

    return $phone;

}

```

  

#### Formatação da Mensagem SMS

  

A classe de notificação `ShortMessageService` formata e envia o SMS:

```php

public function toVonage(object $notifiable): VonageMessage

{

    // Obtém o remetente do SMS da configuração com o tratamento de tipo adequado

    $smsFrom = (string) (config('services.vonage.sms_from') ?? '');

    // Obtém a referência do cliente com verificação de tipo

    $clientReference = method_exists($notifiable, 'getKey') ?

        (string) $notifiable->getKey() :

        (property_exists($notifiable, 'id') ? (string) $notifiable->id : '');

  

    return (new VonageMessage)

        ->content("Sua senha foi redefinida. Sua nova senha é: \n {$this->newPassword} \n Por favor, altere-a após o primeiro acesso.")

        ->unicode()

        ->clientReference($clientReference)

        ->from($smsFrom);

}

```

  

#### Lógica de Redefinição de Senha

  

A lógica de redefinição de senha no método `handlePhoneRecovery`:

```php

protected function handlePhoneRecovery(array $data): void

{

    // Obtém a classe do modelo User da configuração e busca pelo número de telefone

    $userModelClass = config('auth.providers.users.model', User::class);

    if (!class_exists($userModelClass)) {

        $userModelClass = User::class;

    }

    /** @var User|null $user */

    $user = $userModelClass::where('phone', $data['phone'])->first();

  

    if (!$user) {

        Notification::make()

            ->title('Não encontramos uma conta com este número de telefone.')

            ->danger()

            ->send();

        return;

    }

  

    $newPassword = Str::password(10);

    try {

        $user->password = Hash::make($newPassword);

        $user->must_change_password = true;

        $user->save();

        NotificationFacade::send($user, new ShortMessageService($newPassword));

  

        Notification::make()

            ->title('Sua senha foi redefinida com sucesso!')

            ->body('Uma nova senha foi enviada para seu telefone via SMS.')

            ->success()

            ->send();

    } catch (\Exception $e) {

        Notification::make()

            ->title('Erro ao redefinir senha')

            ->body('Ocorreu um erro ao tentar redefinir sua senha. Detalhes: '.$e->getMessage())

            ->danger()

            ->send();

    }

}

```

  

## Melhorias na Segurança de Tipos

  

Várias melhorias na segurança de tipos foram implementadas para garantir um código robusto:

  

1. Adicionadas anotações PHPDoc adequadas para todos os métodos

2. Adicionadas declarações de tipo estritas para parâmetros e valores de retorno

3. Adicionado tratamento adequado de valores nulos para números de telefone e outros valores potencialmente nulos

4. Adicionados valores de fallback para configurações

5. Adicionado tratamento adequado de erros e feedback ao usuário

  

## Requisitos de Configuração

  

Para usar a funcionalidade de recuperação de senha por SMS, você precisa configurar as credenciais da API Vonage no seu arquivo `.env`

VONAGE_SMS_FROM=99999999999
VONAGE_KEY=***
VONAGE_SECRET=*******
