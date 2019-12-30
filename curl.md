# CURL

### Download

### Post Json Data

```bash
$ curl -X POST -H 'Content-Type: application/json' -d '{"key":"value"}' http://domain/upload
```

### Post Files

```bash
curl -X POST -F 'image=@/path/to/pictures/pic.jpg' http://docmain/upload