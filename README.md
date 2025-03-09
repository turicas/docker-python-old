# Docker Images for Old Python Versions with Newer Debian Base (bookworm)

The [official Python Docker images](https://hub.docker.com/_/python) currently (as of 2025-03-08) offers the following
(for the latest Debian releases):

- `2.7`: `buster`
- `3.6`: `bullseye` (with [this patch](https://github.com/pyenv/pyenv-virtualenv/issues/410#issuecomment-1125942002) to
  avoid segmentation fault when using `pip`)
- For 3.7+, `bookworm` is available

I just got the official Dockerfile for those Python versions and upgraded them to be based on Debian bookworm. You can
use [the following image versions from Docker Hub](https://hub.docker.com/r/turicas/python):

- `turicas/python:2.7-slim-bookworm` (compressed size: 49 MB)
- `turicas/python:3.6-slim-bookworm` (compressed size: 41.75 MB)


## Building

```shell
git clone https://github.com/turicas/docker-python-old docker-python-old
cd docker-python-old

docker build -f py27-slim-bookworm.Dockerfile -t turicas/python:2.7-slim-bookworm . # ~3min40s
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
upgraded to Debian bookworm. The process can be automated by running:

```
git clone https://github.com/turicas/docker-python-old docker-python-old
git clone https://github.com/docker-library/python.git docker-python

cd docker-python

# Get latest Python 3.6 Dockerfile available
removal_commit=$(git log --oneline -- 3.6 | cut -d ' ' -f 1 | head -1)
git checkout ${removal_commit}~1
cp 3.6/bullseye/slim/Dockerfile ../docker-python-old/py36-slim-bullseye.Dockerfile
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
' py27-slim-buster.Dockerfile > py27-slim-bookworm.Dockerfile

rm py36-slim-bullseye.Dockerfile py27-slim-buster.Dockerfile
```
