# Docker Images for Old Python Versions with Newer Debian Base (bookworm)

The [official Python Docker images](https://hub.docker.com/_/python) currently (as of 2025-03-08) have the following
Debian releases:

- `2.7`: `buster`
- `3.5`: `buster`
- `3.6`: `bullseye`
- `3.7+`: `bookworm`

I needed a more up-to-date Debian version but running these old Python versions, so I got the official Python
Dockerfiles and upgraded them to be based on Debian bookworm. You can use [the following image versions
from Docker Hub](https://hub.docker.com/r/turicas/python):

- `turicas/python:2.7-slim-bookworm` (compressed size: 49 MB, uncompressed: 139 MB)
- `turicas/python:3.5-slim-bookworm` (compressed size: 46 MB, uncompressed: 131 MB)
- `turicas/python:3.6-slim-bookworm` (compressed size: 42 MB, uncompressed: 120 MB)

> Note: you may want to just use [pyenv](https://github.com/pyenv/pyenv/) instead of those images. I needed to have an
> isolated environment as close as possible to the official releases, so I reused the Dockerfiles. The only patch
> applied was to make pip usable - you may want to apply other [pyenv
> patches](https://github.com/pyenv/pyenv/tree/master/plugins/python-build/share/python-build/patches).


## Building

```shell
git clone https://github.com/turicas/docker-python-old docker-python-old
cd docker-python-old

docker build -f py27-slim-bookworm.Dockerfile -t turicas/python:2.7-slim-bookworm . # ~3min40s
docker build -f py35-slim-bookworm.Dockerfile -t turicas/python:3.5-slim-bookworm . # ~3min37s
docker build -f py36-slim-bookworm.Dockerfile -t turicas/python:3.6-slim-bookworm . # ~2min36s
```

Result:

```
$ docker run --rm turicas/python:2.7-slim-bookworm bash -c 'python --version; cat /etc/debian_version'
Python 2.7.18
12.9

$ docker run --rm turicas/python:3.6-slim-bookworm bash -c 'python --version; cat /etc/debian_version'
Python 3.6.15
12.9
```


## Creating Dockerfiles

The Dockerfiles were based on the [official Python Dockerfiles](https://github.com/docker-library/python) and
upgraded to Debian bookworm. Some things need to be changed/updated in the original Dockerfiles:
- (all) Upgrade Debian version
- (all) Fix Dockerfile `ENV` declarations (add `=` between key and value) to avoid warnings
- (all) Rename variable `GPG_KEY` to `GPG_PUBKEY` to avoid Docker complaining about stolen keys
- (py35, py36) Apply [this patch](https://github.com/pyenv/pyenv-virtualenv/issues/410#issuecomment-1125942002) to fix
  alignment and avoid segmentation fault when using `pip`
- (py27, py35) Change GPG key server to Ubuntu's


The process can be automated by running:

```
git clone https://github.com/turicas/docker-python-old docker-python-old
git clone https://github.com/docker-library/python.git docker-python

cd docker-python

# Get latest Python 3.6 Dockerfile available
removal_commit=$(git log --oneline -- 3.6 | cut -d ' ' -f 1 | head -1)
git checkout ${removal_commit}~1
cp 3.6/bullseye/slim/Dockerfile ../docker-python-old/py36-slim-bullseye.Dockerfile
git checkout master

# Get latest Python 3.5 Dockerfile available
removal_commit=$(git log --oneline -- 3.5 | cut -d ' ' -f 1 | head -1)
git checkout ${removal_commit}~1
cp 3.5/buster/slim/Dockerfile ../docker-python-old/py35-slim-buster.Dockerfile
git checkout master

# Get latest Python 2.7 Dockerfile available
removal_commit=$(git log --oneline -- 2.7 | cut -d ' ' -f 1 | head -1)
git checkout ${removal_commit}~1
cp 2.7/buster/slim/Dockerfile ../docker-python-old/py27-slim-buster.Dockerfile
git checkout master

cd ../docker-python-old
sed '
  s/debian:bullseye-slim/debian:bookworm-slim/;
  s/ENV \(.*\?\) \(.*\)$/ENV \1=\2/;
  s/GPG_KEY/GPG_PUBKEY/g;
  /ENV PYTHON_VERSION=.*/a COPY 3.6/alignment.patch /tmp/
  /cd \/usr\/src\/python/a && cat \/tmp\/alignment.patch | patch -fp0 && rm \/tmp\/alignment.patch \\
' py36-slim-bullseye.Dockerfile > py36-slim-bookworm.Dockerfile
sed '
  s/debian:buster-slim/debian:bookworm-slim/;
  s/ENV \(.*\?\) \(.*\)$/ENV \1=\2/;
  s/GPG_KEY/GPG_PUBKEY/g;
  s/--keyserver ha.pool.sks-keyservers.net/--keyserver keyserver.ubuntu.com/;
  /ENV PYTHON_VERSION=.*/a COPY 3.6/alignment.patch /tmp/
  /cd \/usr\/src\/python/a && cat \/tmp\/alignment.patch | patch -fp0 && rm \/tmp\/alignment.patch \\
' py35-slim-buster.Dockerfile > py35-slim-bookworm.Dockerfile
sed '
  s/debian:buster-slim/debian:bookworm-slim/;
  s/ENV \(.*\?\) \(.*\)$/ENV \1=\2/;
  s/GPG_KEY/GPG_PUBKEY/g;
  s/--keyserver ha.pool.sks-keyservers.net/--keyserver keyserver.ubuntu.com/;
' py27-slim-buster.Dockerfile > py27-slim-bookworm.Dockerfile

rm py36-slim-bullseye.Dockerfile py35-slim-buster.Dockerfile py27-slim-buster.Dockerfile
```
