

## 0) Prerrequisitos

### 0.1 Abrir la consola correcta

* Abre: **MSYS2 MinGW x64** (la que pone `MINGW64` en el prompt).
* Evita: `MSYS` a secas (ahí cambian paths y paquetes).

### 0.2 Actualizar MSYS2 y reiniciar la consola

```bash
pacman -Syu
# cierra la ventana y vuelve a abrir "MSYS2 MinGW x64"
pacman -Syu
```

### 0.3 Instalar toolchain y dependencias (lo que nos hizo falta)

```bash
pacman -S --needed \
  mingw-w64-x86_64-toolchain \
  mingw-w64-x86_64-make \
  mingw-w64-x86_64-cmake \
  mingw-w64-x86_64-ninja \
  mingw-w64-x86_64-jsoncpp \
  mingw-w64-x86_64-bzip2 \
  mingw-w64-x86_64-openssl \
  git
```



### 0.4 Verificación rápida (para evitar “no command found”)

```bash
which g++
which gcc
which mingw32-make
which cmake
which ninja
```

---

## 1) Entrar al repo de jitterentropy (ya descargado)

Asumiendo que lo tienes en:
`C:\Users\yo\Documents\entopia\jitterentropy-library`

En MSYS2 (ojo: rutas Windows se ven como `/c/...`):

```bash
cd /c/Users/yo/Documents/entopia/jitterentropy-library
ls
```

Ir a los tests:

```bash
cd tests/raw-entropy
ls
```

---

## 2) Generar muestras raw en Windows (userspace) con `jitterentropy-hashtime`

> Nota: en Windows **lo hicimos en PowerShell**, porque ahí ya te funcionó directo y sin pelearte con includes raros.

### 2.1 PowerShell: compilar `jitterentropy-hashtime.exe` (sin optimizaciones)

Abre **PowerShell** y ve a:

```powershell
cd C:\Users\yo\Documents\entopia\jitterentropy-library\tests\raw-entropy\recording_userspace
```

Compila (IMPORTANTE `-O0`, si no el propio código aborta):

```powershell
gcc -O0 -g `
  -I..\..\..\src -I..\..\..\ `
  jitterentropy-hashtime.c `
  -o jitterentropy-hashtime.exe
```

Crear carpeta de resultados:

```powershell
New-Item -ItemType Directory -Force ..\results-measurements | Out-Null
```

Generar un `.data` (ejemplo: 1,000,000 muestras, 1 fichero, OSR=3):

```powershell
.\jitterentropy-hashtime.exe 1000000 1 ..\results-measurements\jent-raw-noise --osr 3
```

Comprobar que existe:

```powershell
dir ..\results-measurements | Select-Object -First 10
Get-Content ..\results-measurements\jent-raw-noise-0001.data -TotalCount 5
```

---

## 3) Convertir `.data` a `.bin` para SP800-90B (lo que hacíamos en Linux)

En el árbol `tests/raw-entropy` tienes un script `make_bins.py`.

### 3.1 Volver a MSYS2 MINGW64

```bash
cd /c/Users/yo/Documents/entopia/jitterentropy-library/tests/raw-entropy
ls
```

### 3.2 Crear los `.bin` (según tu flujo, acabaste con `noise_ff_8.bin` y `noise_0f_4.bin`)

Ejecuta:

```bash
python make_bins.py
```

Verifica que existen (en tu caso estaban aquí):

```bash
ls -lah results-measurements/*.bin
```

(Deberías ver ficheros tipo `noise_ff_8.bin`, `noise_0f_4.bin`, etc.)

---

## 4) Instalar/compilar la herramienta NIST SP800-90B (C++)

### 4.1 Clonar el repo NIST dentro de `tests/raw-entropy`

```bash
cd /c/Users/yo/Documents/entopia/jitterentropy-library/tests/raw-entropy
git clone https://github.com/usnistgov/SP800-90B_EntropyAssessment.git
```

Entrar:

```bash
cd SP800-90B_EntropyAssessment/cpp
ls
```

### 4.2 Primer intento: falla por `divsufsort.h` (no está en pacman)

Comando:

```bash
mingw32-make non_iid
```

Falla con:

* `fatal error: divsufsort.h: No such file or directory`

Confirmación de que no hay paquete en MSYS2:

```bash
pacman -Ss divsufsort
# (no sale nada relevante)
```

### 4.3 Solución: compilar `libdivsufsort` desde GitHub y “install” en /mingw64

Clonar dentro de `cpp/`:

```bash
cd /c/Users/yo/Documents/entopia/jitterentropy-library/tests/raw-entropy/SP800-90B_EntropyAssessment/cpp
git clone https://github.com/y-256/libdivsufsort.git
cd libdivsufsort
```

Build con CMake+Ninja (tu CMake 4.x exige política mínima para proyectos viejos):

```bash
mkdir -p build
cd build

cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/mingw64 .. -DCMAKE_POLICY_VERSION_MINIMUM=3.5
ninja
ninja install
```

Verificar instalación:

```bash
ls -lah /mingw64/include/divsufsort.h
ls -lah /mingw64/lib | grep -i divsuf
ls -lah /mingw64/bin | grep -i divsuf
```

---

## 5) Ajustes mínimos al Makefile y al código NIST (porque el upstream esperaba divsufsort64)

### 5.1 Volver a `SP800-90B_EntropyAssessment/cpp`

```bash
cd /c/Users/yo/Documents/entopia/jitterentropy-library/tests/raw-entropy/SP800-90B_EntropyAssessment/cpp
```

### 5.2 El NIST tool incluye `divsufsort64.h` pero nosotros solo instalamos `divsufsort.h`

Crear un header “shim” para que compile:

```bash
cat > /mingw64/include/divsufsort64.h <<'EOF'
#pragma once
#include <divsufsort.h>
EOF
```

Verificar:

```bash
ls -lah /mingw64/include/divsufsort64.h
```

### 5.3 Ajustar includes de jsoncpp en el Makefile (en MinGW64 están en `/mingw64/include/jsoncpp`)

Editar Makefile:

```bash
sed -i 's|-I/usr/include/jsoncpp|-I/mingw64/include/jsoncpp|g' Makefile
sed -i 's|CXXFLAGS = |CXXFLAGS = -I/mingw64/include |' Makefile
```

### 5.4 Quitar `-ldivsufsort64` (no existe) y enlazar `-ldivsufsort`

Primero elimina `-ldivsufsort64` si aparece:

```bash
sed -i 's/-ldivsufsort64//g' Makefile
```

Asegura que linka `-ldivsufsort` (en tu flujo lo insertaste justo antes de jsoncpp):

```bash
grep -n "ldivsufsort" Makefile || echo "No -ldivsufsort in Makefile"
sed -i 's/\(-ljsoncpp\)/-ldivsufsort \1/g' Makefile
```

### 5.5 Patch del fichero `shared/lrs_test.h` para no depender de tipos/funciones 64-bit

Comentar el include 64-bit:

```bash
sed -i 's|#include <divsufsort64.h>|// #include <divsufsort64.h>|' shared/lrs_test.h
```

Cambiar tipos y función 64→normal:

```bash
sed -i 's/saidx64_t/saidx_t/g; s/divsufsort64/divsufsort/g' shared/lrs_test.h
```

---

## 6) Compilar `ea_non_iid.exe`

Limpiar y compilar:

```bash
mingw32-make clean
mingw32-make non_iid
```

Ver que existe:

```bash
ls -lah ea_non_iid.exe
```

---

## 7) Ejecutar SP800-90B non-IID sobre tus `.bin`

### 7.1 Ejecutar para 8-bit (tu caso `noise_ff_8.bin`)

```bash
cd /c/Users/yo/Documents/entopia/jitterentropy-library/tests/raw-entropy/SP800-90B_EntropyAssessment/cpp

./ea_non_iid.exe -i -a -v \
  /c/Users/yo/Documents/entopia/jitterentropy-library/tests/raw-entropy/results-measurements/noise_ff_8.bin \
  8 | tee /c/Users/yo/Documents/entopia/ea_non_iid_ff8.txt
```

### 7.2 Ejecutar para 4-bit (tu caso `noise_0f_4.bin`)

```bash
./ea_non_iid.exe -i -a -v \
  /c/Users/yo/Documents/entopia/jitterentropy-library/tests/raw-entropy/results-measurements/noise_0f_4.bin \
  4 | tee /c/Users/yo/Documents/entopia/ea_non_iid_0f4.txt
```

---

## 8) Resultado final (lo que ya viste)

Al final de cada ejecución aparece algo así:

* Para 8-bit:

```
H_original: 6.913958
H_bitstring: 0.906084
min(H_original, 8 X H_bitstring): 6.913958
```

* Para 4-bit:

```
H_original: 3.430474
H_bitstring: 0.885680
min(H_original, 4 X H_bitstring): 3.430474
```

---








































































Si tomas una estimación conservadora (H_{\min,\text{per bit}}\approx 0.8576), entonces:

[
\text{OSR}^* = \left\lceil \frac{1}{H_{\min,\text{per bit}}} \right\rceil
= \left\lceil \frac{1}{0.8576185} \right\rceil
= \lceil 1.166 \rceil = 2
]

O sea: con este runtime test, **OSR=2** ya sería suficiente según esta métrica (y OSR=3 es claramente conservador).


* Usa el valor que te da el tool como “min(…)” y aclara el tamaño de símbolo:

  * (H_{\min}^{(4)} = 3.430474\ \text{bits/symbol}) con 4 bits/símbolo
  * (H_{\min}^{(8)} = 6.913958\ \text{bits/symbol}) con 8 bits/símbolo
* Y añade la normalización:

  * (0.8576\ \text{bits/bit}) (4-bit)
  * (0.8642\ \text{bits/bit}) (8-bit)

Si quieres, el siguiente paso natural para “como el paper” es:

1. repetir exactamente lo mismo para varios `--osr` (p.ej. 1,2,3,4,6),
2. tabular (H_{\min}) (4-bit y 8-bit),
3. elegir el OSR mínimo que te deje (H_{\min,\text{per bit}}\ge 0.5) (o el umbral que definas) y documentarlo.


