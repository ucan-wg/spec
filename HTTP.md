<!-- Appaently the best practice is to break these out into their own repos, but since I'm trying to figure out what breaking these out at all looks like, I'm going to prototype it here first ~expede -->

# UCAN-over-HTTP Transport Specification v0.1.0



## Collection Format

A UCAN collection transmitted over HTTP MUST be formatted as follows:

A JSON object, 

```
{
  "v": "0.1.0",
  "t": "ucan[]",
  "_": "bafy123",
  "bafy123": [
    "eydsmakdsjdlkas",
    "eyjdksajlkdas",
    "987udsijahuiqdkja"
   ]
}
```


## Encoding

### Payload

Each UCAN is an ordered triple of a base64URL header, base64URL body, and base64URL signature.

### Encoding and URL-safety

The container itself is not base64ed


