# auth-chainer

## Podman
`podman machine init --cpus 4 --disk-size 100 -m 4096 --rootful --now`

## Github App
Configure https://private-7f000001.nip.io/oauth2/callback
## Mkcert
https://github.com/FiloSottile/mkcert

```
mkcert -install
```

```
mkcert --cert-file kustomize/certificates/site-certificates.crt \
-key-file kustomize/certificates/site-certificates.key "*.nip.io"
```

`~/.config/containers/containers.conf`

```
[engine]
tlsverify = false
```



# generate cookie secret
dd if=/dev/urandom bs=32 count=1 2>/dev/null | base64 | tr -d -- '\n' | tr -- '+/' '-_'; echo