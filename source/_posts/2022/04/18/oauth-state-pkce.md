---
title: 【筆記】OAuth 2.0 的 `state` 與 PKCE
date: 2022-04-18 21:14:04
tags:
---

前陣子公司需要製作自己的 OAuth Provider，所以看了不少文件，記錄一下跟同事之間切磋、交流後的一些筆記。

## `state`

`state` 在 [RFC 6749 #10.12](https://datatracker.ietf.org/doc/html/rfc6749#section-10.12) 有提到相關用途，主要是為了防範 CSRF。看了滿多網路上的資源都沒有提到相關的攻擊細節，研究了很久，終於有點心得。

### 情境

假設有個網站（`grass-fed.engineer`），可以讓使用者在上面做筆記，使用者可以在登入後，選擇將筆記存入自己的 GitHub (`github.com`) 中。

1. 使用者會先登入 `grass-fed.engineer`，做完筆記，然後按下按鈕將筆記存入 GitHub 中。

2. 因為會需要存取 GitHub 的資源，按下「存入 GitHub」按鈕後，使用者會被導向 GitHub OAuth 授權存取。這時瀏覽器網址會是

    ```
    https://github.com/login/oauth/authorize?redirect_uri=https://grass-fed.engineer/oauth/callback
    ```

3. 使用者在 GitHub 完成授權後，會被導向

    ```
    https://grass-fed.engineer/oauth/callback?code=XMPcVwx8vngm7NhJIAQdQPAwKH2m2YUZ
    ```

4. Client 收到 request，會拿出 `code`，搭配 `client_secret` 到 GitHub 交換 `access_token`。

### 潛在問題

一切都看似美好，但是其實任何人都可以透過讓使用者呼叫 `https://grass-fed.engineer/oauth/callback?code=<壞人的 code>` 來欺騙 client 執行第 4 步。

這樣就有可能使用者綁定的 GitHub 帳號會變成壞人的帳號。

聽起來好像是壞人把自己帳號送了出去，沒什麼好處，但是從此以後，壞人就會拿到所有使用者想存在自己帳號的筆記。

輸出筆記事小，但是如果是 PayPal 存入授權，想來就是會出大事的。

實務上的操作可以是透過網頁的 `img` tag 嵌入壞人的 URL，讓瀏覽器在下載圖片時變成收到 redirect 請求，進而跳到 `grass-fed.engineer` 進行帳號授權綁定壞人的 GitHub 帳號。

### 解方

`state` 是由 client 產生，令人難以捉摸的一串字，通常會被存放在 cookies 或是 local storage 裡，並且會跟隨第 1 步一起送給 Authorization Server。

```
https://github.com/login/oauth/authorize?state=9YVefLNc&redirect_uri=https://grass-fed.engineer/oauth/callback
```

Autorization Server 會在第 3 步重新導向時，在 URI 加入相同的 `state`。

```
https://grass-fed.engineer/oauth/callback?state=9YVefLNc&code=XMPcVwx8vngm7NhJIAQdQPAwKH2m2YUZ
```

Client 在收到 request 後，會驗證 URL 中的 `state` 是否跟 local 的 `state` 相同，如果是，就代表這個 request 當初是自己發出去的，可以開始兌換 `access_token`；如果不相同，就代表有可能被偽冒了，必須終止兌換流程。

### 小結
`state` 可以用來確保 request 確實是在同一個 client 發送的，而不是他人任意給的假冒 request。

## PKCE

PKCE，全名是 Proof Key for Code Exchange，是 OAuth 2.0 的一種擴充協議。主要是為了解決部分 client 無法安全保護 `client_secret` 以及 authorization code 時的一種安全驗證機制。

### 情境

最常見的場景是原生手機 app，使用者在瀏覽器上完成 Authorization Server 授權後，Authorization Server 會重新導向至指定 URL。

這種 URL 的 protocol 通常不是 `http://` 或 `https://`，而是如 `engineer://` 這類的 deep link，這樣一來才能夠將使用者導回 app 中。

這類的 deep link 仰賴手機作業系統處理，使用者的手機極有可能暗藏惡意軟體，會從中攔截 `code`，然後利用 `code` 換取不當授權。

### 解方

原生手機 app 產生 `code_verifier`，並利用特定方式生成 `code_challenge`，詳細內容可以參考 [RFC 7636 #4.1](https://datatracker.ietf.org/doc/html/rfc7636)。

在 URL 中加入特定參數 `code_challenge`，並利用瀏覽器開啟連結進行授權

```
https://github.com/login/oauth/authorize?code_challenge=DU5hBtgoNO7ejhinrnvxNOFvc5JQyA6Ki7MmFsFIdJzViY&redirect_uri=engineer://oauth/callback
```

Autorization Server 完成授權後，會將 `code_challenge` 跟 `code` 存下，之後會導向至

```
engineer://oauth/callback?code=9O5avOd4vlOTY6Ye96JSyRxSIDKGQIzF
```

原生手機 app 開啟，並利用 `code` 以及 `code_verifier` 交換 `access_token`。如果 Authorization Server 上的 `code_challenge` 跟傳送上來的 `code_verifier` 相符，則配發 `access_token`；反之，不配發 `access_token`。

這樣一來即使 `code` 被攔截，也很難真的換到 `access_token`。

### 小結
在沒有辦法安全保存 `client_secret` 的環境中，可以使用 PCKE 產生一次性的驗證金鑰，確保 `code` 不會任意被攔截並換取不屬於自己的 `access_token`。

## 該用什麼？

`state` 跟 PKCE 看似相似，但效用非常不同：

- `state`：防止壞人利用自己的 `code` 讓好人綁錯帳號
- PKCE：防止 `code` 被壞人攔截並挪作他用

最佳作法是兩者皆要實作才可以換取較高的安全性。

## 參考資料
- [Authorization Code Flow with Proof Key for Code Exchange (PKCE)](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-proof-key-for-code-exchange-pkce)
- [Does PKCE replace state in the Authorization Code OAuth flow?](https://security.stackexchange.com/questions/214980/does-pkce-replace-state-in-the-authorization-code-oauth-flow)
