# v3rt3p documentation

Documentation for the self-hosted v3rt3p voice assistant stack, built with [DiploDoc](https://diplodoc.com/).

## Local preview

```sh
rm -rf build
npx -y @diplodoc/cli -i ./ -o ./build
npx -y http-server ./build --port=5000 --host=127.0.0.1 --cors
```

Open http://127.0.0.1:5000/en/ or http://127.0.0.1:5000/ru/.
