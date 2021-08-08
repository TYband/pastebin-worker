# Pastebin based on Cloudflare Workers

This is a pastebin that can be deployed on Cloudflare workers. Try it on [shz.al](https://shz.al). 

**Philosophy**: effortless deployment, friendly CLI usage, rich functionality. 

**Features**:

1. Share your paste with as short as 4 characters
1. Customize the paste URL as you want
1. Make changes to uploaded paste
1. Delete your paste after uploading
1. Let your paste deleted from the server after a period of time
1. Syntax highlighting powered by Prism
1. Redirect to custom URL
1. Specify the mimetype when fetching your paste
1. Optional longer paste URL for better privacy

## Usage

You can enjoy part of features directly on the website (such as [shz.al](https://shz.al)). 

It is recommended to directly use the HTTP API. See the chapter [API reference](#api-reference) for details. 

[pb](/scripts) is bash script to make it easier to use on command line. 

## Deploy

You are free to deploy the pastebin on your own domain, if you host your domain on Cloudflare. 

First, install [wrangler](https://github.com/cloudflare/wrangler) and login your Cloudflare account.

Create two KV namespaces in Cloudflare workers (one for production, one for test). Remember their IDs. If you do not need testing, simply create one.

Modify IDs in `wrangler.toml` according your own account information (`account_id`, `zone_id`, `kv_namespaces.id`, `kv_namespaces.preview_id` are what you need to modify). Refer to [Cloudflare doc](https://developers.cloudflare.com/workers/cli-wrangler/configuration) on how to find out these parameters.

Deploy!

```shell
$ yarn install
$ yarn build
$ wrangler publish
```

## API Reference

### GET `/`

Return the index page. 

### **GET** `/<name>[.<ext>]`

Fetch the paste with name `<name>`. By default it will return the raw content of the paste.  The `Content-Type` header is set to `text/plain;charset=UTF-8`. If `<ext>` is given, the worker will infer mime-type from `<ext>` and change `Content-Type`. This method accepts the following query string parameters: 

- `?lang=<lang>`: optional. returns a web page with syntax highlight powered by prism.js. 

- `?mime=<mime>`: optional. specify the mime-type, suppressing the effect of `<ext>`. Not effect if `lang` is specified (in which case the mime-type is always `text/html`). 

Examples: `GET /abcd?lang=js`, `GET /abcd?mime=application/json`. 

If error occurs, the worker returns status code different than `200`: 

- `404`: the paste of given name is not found. 
- `500`: unexpected exception. You may report this to the author to give it a fix. 

Usage example: 

```shell
$ curl https://shz.al/i-p-
https://web.archive.org/web/20210328091143/https://mp.weixin.qq.com/s/5phCQP7i-JpSvzPEMGk56Q

$ curl https://shz.al/~panty.jpg | feh -

$ firefox 'https://shz.al/kf7z?lang=nix'

$ curl 'https://shz.al/~panty.jpg?mime=image/png' -w '%{content_type}' -o /dev/null -sS
image/png;charset=UTF-8
```

### GET `/u/<name>`

Redirect to the URL recorded in the paste of name `<name>`. 

If error occurs, the worker returns status code different than `301`: 

- `404`: the paste of given name is not found. 
- `500`: unexpected exception. You may report this to the author to give it a fix. 

Usage example: 

```shell
$ firefox https://shz.al/u/i-p-

$ curl -L https://shz.al/u/i-p-
```

### **POST** `/`

Upload your paste. It accept parameters in form-data: 

- `c`: mandatory. The **content** of your paste, text of binary. It should be no larger than 10 MB. 

- `e`: optional. The **expiration** time of the paste (in seconds). After this period of time, the paste is permanently deleted. It should be no smaller than 60 due to the limitation if Cloudflare KV storage. 

- `s`: optional. The **password** which allows you to modify and delete the paste. If not specified, the worker will generate a random string as password. 

- `n`: optional. The customized **name** of your paste. If not specified, the worker will generate a random string (4 characters by default) as the name. You need to prefix the name with `~` when fetching the paste of customized name. The name is at least 3 characters long, consisting of alphabet, digits and characters in `+_-[]*$=@,;`. 

- `h`: optional. The flag of **human friendly output**. If specified to any value, it will return a web page instead of JSON. 

- `p`: optional. The flag of **private mode**. If specified to any value, the name of the paste is as long as 24 characters. No effect if `n` is used. 

`POST` method returns a JSON string by default, if no error occurs, example: 

  ```json
  {
      "url": "https://shz.al/abcd", 
      "admin": "https://shz.al/abcd:w2eHqyZGc@CQzWLN=BiJiQxZ",
      "expire": 100,
      "isPrivate": false,
  }
  ```

  Explanation of the fields:

  - `url`: mandatory. The URL to fetch the paste. When using a customized name, it looks like `https//shz.al/~myname`. 
  - `admin`: mandatory. The URL to update and delete the paste, which is `url` suffixed by `~` and the password. 
  - `expire`: optional. The expiration seconds. 
  - `isPrivate`: mandatory. Whether the paste is in private mode. 

If error occurs, the worker returns status code different than `200`: 

- `400`: your request is in bad format. 
- `409`: the name is already used. 
- `413`: the content is too large. 
- `500`: unexpected exception. You may report this to the author to give it a fix. 

Usage example: 

```shell
$ curl -Fc="kawaii" -Fe=300 -Fn=hitagi https://shz.al  # uploading some text
{
  "url": "https://shz.al/~hitagi",
  "admin": "https://shz.al/~hitagi:22@-OJWcTOH2jprTJWYadmDv",
  "isPrivate": false,
  "expire": 300
}

$ curl -Fc=@panty.jpg -Fn=panty -Fs=12345678 https://shz.al   # uploading a file
{
  "url": "https://shz.al/~panty",
  "admin": "https://shz.al/~panty:12345678",
  "isPrivate": false
}
```

### **PUT** `/<name>:<passwd>`

Update you paste. It accept the same parameters in form-data: 

- `c`: mandatory. Same as `POST` method. 
- `e`: optional. Same as `POST` method. Note that the deletion time is now recalculated. 
- `s`: optional. Same as `POST` method. 
- `h`: optional. Same ss `POST` method. 

The returning of `PUT` method is also the same as `POST` method. 

If error occurs, the worker returns status code different than `200`: 

- `400`: your request is in bad format. 
- `403`: your password is not correct. 
- `404`: the paste of given name is not found. 
- `413`: the content is too large. 
- `500`: unexpected exception. You may report this to the author to give it a fix. 

Usage example: 

```shell
$ curl -X PUT -Fc="kawaii~" -Fe=500 https://shz.al/~hitagi:22@-OJWcTOH2jprTJWYadmDv
{
  "url": "https://shz.al/~hitagi",
  "admin": "https://shz.al/~hitagi:22@-OJWcTOH2jprTJWYadmDv",
  "isPrivate": false,
  "expire": 500
}

$ curl -X PUT -Fc="kawaii~" https://shz.al/~hitagi:fGsQ@SkGAcmVJHcWgKABNsYK
{
  "url": "https://shz.al/~hitagi",
  "admin": "https://shz.al/~hitagi:fGsQ@SkGAcmVJHcWgKABNsYK",
  "isPrivate": false
}
```

### DELETE `/<name>:<passwd>`

Delete your paste. It may takes seconds to synchronize the deletion globally. 

If error occurs, the worker returns status code different than `200`: 

- `403`: your password is not correct. 
- `404`: the paste of given name is not found. 
- `500`: unexpected exception. You may report this to the author to give it a fix. 

Usage example: 

```shell
$ curl -X DELETE https://shz.al/~hitagi:fGsQ@SkGAcmVJHcWgKABNsYK
the paste will be deleted in seconds

$ curl https://shz.al/~hitagi
not found
```
