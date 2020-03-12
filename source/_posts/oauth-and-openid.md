---
title: OAuth与OpenID
tags:
  - web
  - security
date: 2020-03-12 22:12:31
---

最近经常遇到客户端和第三方H5之间身份互认的场景，打算在这里总结一下OAuth和OpenID这两个“以为自己了解其实自己并不了解”的知识点。
<!--more-->

## OAuth 2.0

[OAuth](https://oauth.net/)是一个公开的安全授权协议，目前最新版本为OAuth 2.0，对应场景是为第三方应用对当前应用的特定操作授权，这里的操作一般是访问用户的账户/身份信息。

一次OAuth流程包括四个角色，分别为发起授权请求的第三方应用、响应授权请求的授权服务、支持用户信息访问操作的资源服务、用户本身即用户信息拥有者和授权请求的决定方。大致来说，首先使用OAuth的第三方应用需要预先在OAuth授权服务上进行注册/登记，授权服务会为该应用分配一个应用id和应用密钥secret（用于构建授权请求与识别应用），注意第三方应用还需要注册回调URI，授权服务只会向已注册的回调URI传递授权相关信息。然后该应用发起授权请求获取用户授权，从授权服务获取对应**access token**，就可以使用该access token通过资源服务来访问相应资源。

可以看出OAuth的核心在于两个交互过程，一个是第三方应用与授权服务的授权交互（获取token），还有一个是第三方应用使用access token与资源服务的交互（使用token），权限控制集中在token上，将操作权限封装委托到access token上可以增强对操作权限的管理和控制能力，同时可以减少密码等敏感信息泄露的风险。

对于客户端应用和授权服务之间的授权交互主要有两种适用模式：

### 授权码模式

适用于有服务端支持（安全存放应用密钥secret、接收重定向授权码请求）的前端应用（server-side app）。授权码模式将获取token拆成两步完成，第一步前端界面构造授权请求跳转授权服务页，用户授权完成后授权服务向重定向URI传递一个授权码而不是最终token，然后应用需要继续使用该授权码（并附带应用密钥secret）向授权服务请求最终的token。

注意两步请求的重定向URI必须相同，同时第一步请求时需要带上一个随机串state，授权服务会将该随机串也通知到重定向URI,便于重定向endpoint核实。

整体流程如下：


    第三方应用 -(生成随机state，用state、应用id、重定向URI发起授权码请求，跳转到授权服务页，关联state和请求cookie)->
    
    授权服务页 -(客户完成授权，核实应用id、重定向URI一致，重定向传递授权码、state)->
    
    重定向URI -(获取授权码，核实state与cookie一致，服务端用应用id、应用密钥secret、授权码、重定向URI发起token获取请求)->
    
    授权服务 -(核实请求里的应用id、应用密钥secret、授权码、重定向URI一致，返回token)->
    
    应用获取token

可以看出安全风险在于可能的授权码泄露和token泄露。由于获取授权码阶段需要客户和授权服务交互确认授权，此阶段一般是在浏览器环境中完成的，授权服务通过浏览器重定向传递授权码这一步相对更容易被攻击。

获取token的互信主要依赖于：

1. 授权码与上一阶段授权服务返回的一致，表示（正常情况下）这次请求的发起方刚利用受信任的重定向URI获取到授权码，请求方身份的可信度较大；
2. 应用id、应用密钥secret、重定向URI参数一致，尤其是（正常情况下）安全存放的应用密钥secret一致，这基本可以核实请求发起方身份，但是由于获取授权码和获取token两步只依靠授权码进行安全关联（应用id和重定向URI为固定值且较容易暴露），如果存在应用secret泄露的情况，那只要攻击者拦截到授权码就能伪造token请求轻松获取token；

### 授权码PKCE扩展模式

适用于无法安全存放应用密钥secret的前端应用（public client），如单页web应用或者原生移动应用，同时这种场景重定向URI被拦截导致授权码泄露的风险也更高。KCE扩展提出在授权码模式的基础上，客户端每次请求授权时生成一个随机串来替代secret，这个随机串称为code verifier。应用利用code verifier作哈希生成一个code challenge，并在请求授权码时附带上code challenge。授权服务照常重定向传递授权码，同时将授权码与code challenge关联。然后客户端利用code verifier和授权码来请求token。授权服务对比收到的code verifier与关联的challenge是否一致，以决定是否发放token。整体流程如下：

	第三方应用 -(生成随机state，生成随机code verifier以及对应code challenge，用challenge、state、应用id、重定向URI发起授权码请求，跳转到授权服务页，关联state和请求cookie)->
    
    授权服务页 -(完成授权，核实应用id、重定向URI一致，关联授权码与code challenge，重定向传递授权码、state)->
    
    重定向URI -(获取授权码，核实state与cookie一致，用verifier、应用id、授权码、重定向URI发起token获取请求)->
    
    授权服务 -(核实应用id、verifier、授权码、重定向URI一致，返回token)->
    
    应用获取token

授权码PKCE扩展模式在每次授权请求流程引入随机值对challenge和verifier，分别用于授权码请求和token请求，challenge和verifier之间通过哈希算法单向关联(verifier -hash-> challenge)，加强了获取授权码和获取token两次请求的安全关联性。

和标准授权码模式相比，PKCE扩展模式的互信差异在于：

1. 请求token阶段缺少了应用密钥secret认证，请求方身份可信度有所下降，考虑到参与请求的应用id、重定向URI都为固定值且较容易暴露到前端，可以说请求方身份完全由“请求方通过受信任的重定向URI获取到正确授权码”的事实来认定；
2. 在同一个授权流程中，获取授权码和获取token利用随机生成的challenge-verifier对进行安全关联，且单向哈希算法保证即使拦截到challenge值也无法确定verifier值，这可以协助核实授权码请求和token请求由同一个应用发起，即使恶意攻击者拦截到授权码也无法伪造请求来获取token；

分析下来，PKCE扩展引入的随机challenge-verifier对和应用密钥secret互信并不冲突，似乎在授权码模式也可以加入随机challenge-verifier关联对来增强安全性。

## OpenID

初次接触OpenID是还没毕业那会在某厂实习的时候，当时平台组开发维护了一个基于Python的PaaS平台，内部使用来开发部署一些内部web应用，这些内部应用当然没有必要各自维护自己的一套用户管理体系，所以这里利用了OpenID提供统一identity服务。

大概的操作流程是，应用构造一个OpenID consumer，指定provider URI、重定向URI和需要获取的信息（用户名、邮箱等），然后跳转到provider URI完成登录，登录完成后将登录信息传回到重定向URI，应用确认用户已通过身份认证，然后才允许用户正常使用。可以看出这个认证流程和后来的OAuth 2.0的获取授权码阶段十分相似。

目前OpenID协议最新版本为OpenID Connect，具体流程为基于OAuth 2.0协议来获取和传递identity token，用户身份认证依赖于identity token。抛开authorization和authentication的字面游戏，OAuth 2.0协议本质上利用基于code exchange的两步请求流程来提供安全传递授权信息给请求者的能力，这个授权信息在OAuth 2.0里是access token，而OpenID connect的区别则在于其利用这个两步请求流程来实现安全传递identity token给请求者。

和OAuth中作为权限令牌的access token不同，identity token的语义是一种由OpenID Provider签发的具有自描述信息的身份验证声明，表示形式为JSON Web Token。一个identity token主要包含以下信息：

1. 签发者标识；
2. 供客户端使用的唯一身份标识；
3. identity token的签发受众；
4. 签发时间和过期时间；

除了登录之外，identity token还可以用于无状态会话管理、传递身份信息、交换(access)token等场景。

最后再看看JWT(JSON Web Token)，JWT是一种开放的安全标准，利用经加密签名加持的JSON提供紧凑和自描述的安全传输。JWT的关键在于签发者利用私钥对需要验证的信息加密生成额外签名，在私钥没有泄露的前提下，客户端再次上送JWT时，签发者可以利用验签对信息进行校验确保信息/状态一致。JWT的自描述特性可以使得签发者(服务端)不用维护JWT信息的对应状态，本质上是利用额外计算(验签/加解密)来优化了存储(会话状态维护)的工作。


