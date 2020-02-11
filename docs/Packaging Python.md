---
Author: Mateo Upegui Borja
Company: SETI SA
Dependency: DevOps
---

# Py RESTful
Como diseñar APIs REST con Python, desde cero – Parte 1

## Introducción
Actualmente en la compañía el workflow (flujo de trabajo) que tenemos a la hora de crear aplicaciones Python es menos que ideal, creamos nuestros módulos .py y nuestro archivo requirements.txt y nos desentendemos del proceso de despliegue.

Pero esté estilo de trabajo requiere que la persona encargada del despliegue tenga por obligación que entender de Python, para que pueda instalar la versión de Python adecuada y las dependencias, e incluso no tenemos en cuenta fijar una versión de Python en los metadatos de nuestras aplicaciones abriendo la posibilidad a incompatibilidades.

En palabras de la [Python Packaging Authority](https://packaging.python.org/overview/#python-modules) 
> El patrón de la utilización de modulos _.py_ de python como medio para distribuir proyectos no es escalable para proyectos que consistan de multitudes de ficheros, necesiten librerias adicionales o requieran una versión especifica de python.

## Empaquetando

> El empaquetamiento se puede entender como el arte producir un artefacto contenido para la transferencia y uso confiable en ambientes productivos - [The Packaging Gradient](https://www.youtube.com/watch?v=iLVNWfPWAC8)


A la hora de distribuir nuestros artefactos tenemos muchas alternativas dependiendo de el nivel de portabilidad que queramos tener, es posible utilizar cualquiera de las estrategias mencionadas en [The Packaging Gradient](https://www.youtube.com/watch?v=iLVNWfPWAC8)

![Simple packaging](./img/py_pkg_tools_and_libs.png)

![Advanced packaging](./img/py_pkg_applications.png)

Y se debe escoger la que mejor se ajuste a nuestras necesidades.

## PyPI & PyPa
La __PyPa__ (Python Packaging Authority) publica constantemente una guía de empaquetado de aplicaciones pyton en packaging.python.com, ellos son quienes documentan el proceso ideal de empaquetamiento y son quienes administrar el _Package Index_ (_PyPi_) de donde descargamos nuestros paquetes con _pip_.

Existe una documentación extensiva alrededor de los paquetes en el sitio de la _Python Packaging Authority_ [Packaging - Thinking About Deployment](https://packaging.python.org/overview/#thinking-about-deployment), [Packaging and Distributing projects](https://packaging.python.org/guides/distributing-packages-using-setuptools/).

## ¿Por qué wheels? (Binary Distributions)

Las distribuciones binarias son archivos empaquetados y comprimidos en un archivo con extension _.whl_ (wheel) con el código fuente de nuestra aplicación en python junto a librerias binarias compiladas que necesita, al lado de metadatos que contienen la información de las dependencias, la compatibilidad con versiones de python y otros datos útiles a la hora de instalar y ejecutar el paquete.

Escogeremos esta _"estrategia"_ de empaquetado porque no queremos ir demasiado lejos, no queremos distribuir nuestra aplicación a consumidores sino a administradores de sistemas, además podemos asegurar de manera sencilla que la versión-versiones de python compatibles con nuestro paquete estarán disponibles en la maquina en la que haremos el despliegue.

La solución será muy similar a como en el ecosistema de Java una cantidad considerable de empresas utilizan repositorios de Maven privados para distribuir sus aplicaciones y además será compatible con sistemas de distribución de artefactos como _JFrog Artifactory_ que [tiene soporte para repositorios privados _Pypi_](https://www.jfrog.com/confluence/display/RTF/PyPI+Repositories), [_Package Cloud_](https://packagecloud.io/l/supported-packages) o alternativas gratuitas como [_Pulp_](https://pulpproject.org/).

## Estructura de un paquete Python
Todos los paquetes de python, a la hora de ser codificados siguen una estructura base muy sencilla; principalmente debe existir una carpeta con el nombre del paquete y en esta debe existir un archivo `__init__.py` y usualmente un archivo `setup.py`

```bash
# Estructura de un paquete de python
└───setup.py
└───Foo/
    └───__init__.py
```
Todos los miembros como variables o funciones que declaremos o importemos en el archivo `__init__.py` estarán disponibles al importar nuestro paquete, por ejemplo:

```python
# ./Foo/__init__.py

foo = 'bar'

def func(x):
    return('func' + x)
```
Estarán diponible para otros paquetes al importar `foo` como `foo.foo` y `foo.func()`.

El archivo `setup.py` es un archivo estandar donde se especifican todos los metadatos del paquete como sus dependencias, la documentación de como debería ser un archivo `setup.py` está disponible en [PyPa](https://packaging.python.org/guides/distributing-packages-using-setuptools/#setup-args). Por ejemplo:
```python
# ./setup.py
"""
Foo
-------------
Documentation
"""

from setuptools import setup, find_packages

with open("README.md", "r") as fh:
    long_description = fh.read()

setup(
    name='Foo',
    version='0.1.0',
    url='https://example.com',
    license='BSD',
    author='John Doe',
    author_email='user@example.com',
    description='Package description',
    long_description=long_description,
    long_description_content_type="text/markdown",
    packages=find_packages(),
    zip_safe=False,
    include_package_data=True,
    platforms='any',
    python_requires=">= 3.6",
    install_requires=[
        'Flask',
        'pymongo'
    ],
    classifiers=[
        'Environment :: Web Environment',
        'Intended Audience :: Developers',
        'License :: OSI Approved :: BSD License',
        'Operating System :: OS Independent',
        'Programming Language :: Python',
        'Topic :: Internet :: WWW/HTTP :: Dynamic Content',
        'Topic :: Software Development :: Libraries :: Python Modules'
    ]
)
```

Al cumplir estás dos condiciones ya tendremos un paquete válido a los ojos de la _Packaging Authority_ y ya podemos pasar a empaquetar nuestro paquete y si deseamos subirlo a un package index público o privado. 

Al importar setuptools ya podremos empaquetar nuestra aplicación de manera sencilla como `bdist_wheel` o como `sdist` ejecutando el archivo `setup.py` y añadiendo algunos parametros:

```bash
# Wheel - Binary distribution
python setup.py bdist_wheel

# Source distribution
python setup.py sdist
```

Esto crea una carpeta `./dist` donde reposarán todos los ficheros de nuestra distribución, además crea archivos utilitarios utilizados en la compilación de librerias y en el empaquetamiento de nuestra aplicación, podemos añadir un comando a `setup.py` para encargarnos de esos archivos molestos de la siguiente  manera:

```python
"""
Foo
-------------
Documentation
"""

from setuptools import setup, find_packages, Command
from os import path, remove
from glob import glob
from shutil import rmtree

with open("README.md", "r") as fh:
    long_description = fh.read()

class CleanCommand(Command):
    user_options = []
    
    def initialize_options(self):
        pass
    def finalize_options(self):
        pass
    def run(self):
        _globs = ['./build', './*.pyc', './*.tgz', './*.egg-info']
        for _glob in _globs:
            for _match in glob(_glob):
                if (path.isfile(_match)):
                    print(' - Deleting', _match, 'file') 
                    remove(_match)
                else:
                    print(' - Deleting', _match, 'directory')
                    rmtree(_match)
        print(' - Done')        

setup(
    name='Foo',
    version='0.1.0',
    url='https://example.com',
    license='BSD',
    author='John Doe',
    author_email='user@example.com',
    description='Package description',
    long_description=long_description,
    long_description_content_type="text/markdown",
    packages=find_packages(),
    zip_safe=False,
    include_package_data=True,
    platforms='any',
    python_requires=">= 3.6",
    install_requires=[
        'Flask',
        'pymongo'
    ],
    classifiers=[
        'Environment :: Web Environment',
        'Intended Audience :: Developers',
        'License :: OSI Approved :: BSD License',
        'Operating System :: OS Independent',
        'Programming Language :: Python',
        'Topic :: Internet :: WWW/HTTP :: Dynamic Content',
        'Topic :: Software Development :: Libraries :: Python Modules'
    ],

    cmdclass = {
        'clean': CleanCommand,
    }
)
```

Así cada podemos borrar de manera sencilla todos los archivos intermedios que se crearon en el proceso llamando `python setup.py clean`.

Si deseamos subir nuestro paquete al indice de paquetes público _PyPi_  podemos hacerlo utilizando el paquete/herramienta de linea de comandos `twine`, para ello primero nos registramos en pypi.org/account/register, verificamos nuestro correo y luego invocamos twine y seguimos las instrucciones en la linea de comandos.

Debemos tener especial atención con el nombre de nuestro paquete, ya que si existe algún otro paquete con el mismo nombre no podremos subirlo, por otro lado, y como deberiamos suponer, subir paquetes basura a _PyPi_ es poco recomendado.

```bash
# Instalamos twine
pip install twine


# Subimos a PyPi
twine upload dist/*
```

Para nuestra suerte tambien existe un repositorio de pruebas donde podemos validar el proceso en test.pypi.org, seguimos el mismo proceso registrandonos, pero ahora en test.pypi.org/account/register y ejecutamos el siguiente comando para subir nuestro paquete y seguimos las instrucciones en la linea de comandos

```bash
twine upload --repository-url https://test.pypi.org/legacy/ dist/*
```

Una vez hallamos haya terminado el proceso nuestro paquete estará disponible en el repositorio al que lo hayamos subido y lo podremos descargar con pip.

![Uploaded package](./img/test_pypi_uploaded.png)

O podemos crear nuestro propio repositorio de paquetes y subirlo allí especificando `--repository-url` en las opciones de `twine`.

## Usando Poetry

Como pudimos apreciar, el proceso de administrar un paquete de python mediante el archivo `setup.py` es muy engorroso, es necesario añadir las dependencias manualmente y tambien crear los venvs de manera virtual. Para nuestra suerte existe __Poetry__.

 [Poetry](https://github.com/python-poetry/poetry) es una herramienta de linea de comandos que nos permite administrar dependencias python, pero además nos permite hacer build de nuestro paquete de una manera sencilla.

Para instalarlo en Linux podemos hacerlo ejecutando

```bash
curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python3
```
> ⚠️ Es necesario correr la instalación con python3 

Una vez instalado podemos crear un nuevo projecto de una manera muy parecida al uso de `ng new` (_Angular_) con `poetry new <my-project-name>` 

La lista completa de los comandos que se pueden utilizar en poetry està disponible en la [documentación](https://python-poetry.org/docs/cli/), entre otras cosas se pueden instalar paquetes que se agregarán automaticamente a las dependencias de nuestra aplicación, se pueden instalar dependencias de desarrollo y se puede empaquetar nuestra aplicación con un solo comando entre otras cosas.

## Creando nuestro repositorio de PyPi y configurandolo en Poetry

Podemos crear un indice de paquetes sencillo y open source para distribuir nuestros paquetes y aplicaciones python internamente en nuestra compañía o bien podemos utilizar una solución paga como _JFrog Artifactory_. 

En esta ocasión pondremos en marcha un servidor abierto y gratuito para almacenar nuestros paquetes de python con _Pulp_. Para ello vamos a utilizar un contenedor de _Docker_ 🐳 con el fin de evitar tediosas instalaciónes. Para ello seguimos las instrucciones en la [documentación](https://pulpproject.org/2015/05/21/use-docker-to-try-pulp/).

```bash
wget https://raw.githubusercontent.com/pulp/packaging/docker/dockerfiles/centos/start.sh
mkdir ~/pulp-data
chmod u+x ./start.sh
sudo ./start.sh ~/pulp-data
```

> ⚠️ No se debe containerizar _Pulp_ en producción utilizando este script.

