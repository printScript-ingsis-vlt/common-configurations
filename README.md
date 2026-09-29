## RAZON DE SER
Este repositorio encargado de centralizar los workflows comunes entre los diferentes repositorios de la organización, buscando reducir la replicacion de codigo equivalente entre repositorios

## COMO FUNCIONA

Define los workflows comunes entre repositorios, ya sea solo para uno, para dos o para todos, pues la decision de utilizarlos dependera luego de la implementacion de los mismos en cada microservicio

## COMO UTILIZARLO EN UN MICROSERVICIO

Se instanciaria como algo asi:

```
name: Check

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    uses: common-configurations/.github/workflows/<nombre-del-archivovich>.yml@latest
```
