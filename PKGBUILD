# Maintainer: wabi <aschrafl@jetnet.ch>
# Maintainer: pikl <me@pikl.uk>
pkgbase=immich
pkgname=('immich-server' 'immich-cli')
pkgrel=1
pkgver=1.122.3
pkgdesc='Self-hosted photos and videos backup tool'
url='https://github.com/immich-app/immich'
license=('MIT')
arch=(x86_64)
# ts-node required for CLI
makedepends=('git' 'npm' 'jq' 'python-poetry' 'ts-node')
# combination of server/CLI deps, see split package functions
# for individual deps and commentary
depends=('redis' 'postgresql' 'nodejs>=20'
    'pgvecto.rs=0.2.0' 'zlib' 'glib2' 'expat' 'librsvg' 'libexif'
    'libwebp' 'libjpeg-turbo' 'libgsf' 'libpng'
    'libheif' 'lcms2' 'mimalloc' 'openjpeg2'
    'openexr' 'liblqr' 'libtool' 'jellyfin-ffmpeg'
    'libvips>=8.14.3' 'openslide' 'poppler-glib' 'imagemagick'
    'brotli' 'perl-io-compress-brotli' 'libraw' 'libde265' 'dav1d'
    'highway'
)
source=("${pkgbase}-${pkgver}.tar.gz::https://github.com/immich-app/immich/archive/refs/tags/v${pkgver}.tar.gz"
        "base-images::git+https://github.com/immich-app/base-images"
        "${pkgbase}-server.service"
        "${pkgbase}-machine-learning.service"
        "${pkgbase}.sysusers"
        "${pkgbase}.tmpfiles"
        'immich.conf'
        'nginx.immich.conf'
        # TODO at the moment, the latest version at install will be taken 
        # mirroring approach in docker base-image, however should we implement 
        # a simple service to keep these up-to-date since they appear to be
        # generated daily?
        'https://download.geonames.org/export/dump/cities500.zip'
        'https://download.geonames.org/export/dump/admin1CodesASCII.txt'
        'https://download.geonames.org/export/dump/admin2Codes.txt'
        'https://raw.githubusercontent.com/nvkelso/natural-earth-vector/v5.1.2/geojson/ne_10m_admin_0_countries.geojson')
sha256sums=('7f59cfd085953c0771e7cdb12f563d351a22697ed96daac50a840ca65558b220'
            'SKIP'
            '48ba0c1716e4459322f878775bd37d9f8efe80b9c8a830bdb901ee4cba15a402'
            '637e886cfb5a47834560b00158affe1e218a84e3f825d28d2640c10d2d597ef1'
            '01707746e8718fe169b729b7b3d9e26e870bf2dbc4d1f6cdc7ed7d3839e92c0e'
            '4ae8a73ccbef568b7841dbdfe9b9d8a76fa78db00051317b6313a6a50a66c900'
            '077b85d692df4625300a785eed1efdc7af8fbb8e05dfa8c7d8b4053c1eb76a58'
            '28081014a8c410010df22e979c49c3af4a49815e2c6e4ee6090206e2381a56c6'
            'SKIP'
            'SKIP'
            'SKIP'
            'SKIP')
_installdir=/opt/immich-machine-learning
_venvdir="${_installdir}/venv"

prepare() {
    cd "${srcdir}/${pkgbase}-${pkgver}"

    # TODO possibly can checkout using date directly (commits are tagged with date)
    imgdate=$(grep ^'FROM ghcr.io/immich-app/base-server-prod' server/Dockerfile | cut -d: -f2 | sed 's/^\(.\{4\}\)\(.\{2\}\)/\1-\2-/')
    cd "${srcdir}/base-images"
    imgcommit=$(git log -n1 --no-color --before="$1" | head -n1 | cut -d' ' -f2)
    git checkout "$imgcommit"
    
}

build() {
 
    # build server
    # from: server/Dockerfile RUN npm commands
    #   * npm link / and cache clean not required
    cd "${srcdir}/${pkgbase}-${pkgver}/server"
    npm ci
    npm run build
    tmpdir=$(mktemp -d)
    cp -r node_modules/@img "${tmpdir}"
    cp -r node_modules/exiftool-vendored.pl "${tmpdir}"
    rm -rf "${tmpdir}/@img/sharp-libvips"*
    rm -rf "${tmpdir}/@img/sharp-linuxmusl-x64"
    npm prune --omit=dev --omit=optional
    mkdir -p node_modules/@img node_modules/exiftool-vendored.pl
    mv "${tmpdir}/@img/"* node_modules/@img
    mv "${tmpdir}/exiftool-vendored.pl/"* node_modules/exiftool-vendored.pl
    rm -rf "${tmpdir}"

    # web build
    cd "${srcdir}/${pkgbase}-${pkgver}/open-api/typescript-sdk"
    npm ci
    npm run build

    # build web frontend
    # from: web/Dockerfile RUN npm commands
    cd "${srcdir}/${pkgbase}-${pkgver}/web"
    npm ci
    npm run build
    # npm prune --omit=dev

    # build machine learning (python)
    # from: ENV and RUN commands in machine-learning/Dockerfile
    #   * later ENV commands picked up in systemd service files
    cd "${srcdir}/${pkgbase}-${pkgver}/machine-learning"
    sed -i 's|cache_folder: str = ".*"|cache_folder: str = "/var/lib/immich/.cache"|' app/config.py
    # pip install of poetry not required because poetry is a makedep
    export PYTHONDONTWRITEBYTECODE=1
    export PYTHONUNBUFFERED=1
    export PIP_NO_CACHE_DIR=true
    poetry config installer.max-workers 10
    poetry config virtualenvs.create false
    python -m venv "${srcdir}/venv"
    export VIRTUAL_ENV="${srcdir}/venv"
    export PATH="${srcdir}/venv/bin:${PATH}"
    poetry install --sync --no-interaction --no-ansi --no-root --with cpu --without dev
    # delete any python bytecode 
    find "${srcdir}/venv" -type f -name "*.py[co]" -delete
    find "${srcdir}/venv" -type d -name "__pycache__" -delete
    # relocate without breaking
    sed -i "s|${srcdir}/venv|${_venvdir}|g" "${srcdir}/venv/bin/"* "${srcdir}/venv/pyvenv.cfg"

    # build CLI
    cd "${srcdir}/${pkgbase}-${pkgver}/cli"
    npm ci
    npm run build
}

package_immich-server() {

    replaces=('immich')
    conflicts=('immich')

    # dependencies generated from base-images repository
    # https://github.com/immich-app/base-images/blob/main/server/Dockerfile
    # 1.101.0-2: liborc dep found to be not required
    depends=('redis' 'postgresql' 'nodejs>=20'
        # mirror machine-learning/pyproject.toml requirement
        'pgvecto.rs=0.2.0'  # aur
        'zlib'
        'glib2'
        'expat'
        'librsvg'
        'libexif'
        'libwebp'
        'libjpeg-turbo'
        'libgsf'
        'libpng'
        'libheif'
        'lcms2'
        'mimalloc'
        'openjpeg2'
        'openexr'
        'liblqr'
        'libtool'
        'jellyfin-ffmpeg'  # maintainer advice 28/10/24
        # need to ensure this matches sharp depend version
        # because otherwise a local copy will be built
        # breaking heif conversion
        'libvips>=8.14.3'
        'openslide'
        'poppler-glib'
        'imagemagick'
        'libraw'
        # added v1.108
        'libde265'
        'dav1d'
        # added v1.118
        'brotli'
        'perl-io-compress-brotli'
        # added v1.120.2
        'highway'
    )
    backup=("etc/immich.conf")
    options=("!strip")
    install=${pkgname}.install
    optdepends=(
        'libva-mesa-driver: GPU acceleration'
        'mesa-utils: GPU acceleration'
        'vulkan-driver: Vulkan support'
        'nginx: Reverse proxy'
        'intel-compute-runtime: OpenCL support'
        'intel-media-driver: HW acceleration'
    )

    cd "${srcdir}/${pkgbase}-${pkgver}"
    
    # install server
    # from: server/Dockerfile COPY commands after build
    #   * start*.sh not required
    #   * setting NODE_ENV=production picked up in systemd service file
    install -dm755 "${pkgdir}/usr/lib/immich/app/server"
    cp -r server/node_modules "${pkgdir}/usr/lib/immich/app/server/node_modules"
    cp -r server/dist "${pkgdir}/usr/lib/immich/app/server/dist"
    cp -r server/bin "${pkgdir}/usr/lib/immich/app/server/bin"
    install -Dm644 server/package.json "${pkgdir}/usr/lib/immich/app/server/package.json"
    install -Dm644 server/package-lock.json "${pkgdir}/usr/lib/immich/app/server/package-lock.json"
    install -Dm644 LICENSE "${pkgdir}/usr/lib/immich/app/LICENSE"
    cp -r server/resources "${pkgdir}/usr/lib/immich/app/server/resources"

    # install www
    install -dm755 "${pkgdir}/usr/lib/immich/build"
    cp -r web/build "${pkgdir}/usr/lib/immich/build/www"

    # install machine-learning
    # from: machine-learning/Dockerfile COPY commands
    #   * setting NODE_ENV=production and others picked up in systemd service file
    install -dm755 "${pkgdir}${_installdir}"
    cp -r "${srcdir}/venv" "${pkgdir}${_installdir}"
    cp -r "machine-learning/app" "${pkgdir}${_installdir}"
    cp -r "machine-learning/ann" "${pkgdir}${_installdir}"
    install -Dm644 "machine-learning/log_conf.json" "${pkgdir}${_installdir}/log_conf.json"

    cd "${srcdir}"
    
    # install reverse-geocoding data
    # https://github.com/immich-app/base-images/blob/main/server/Dockerfile
    install -dm755 "${pkgdir}/usr/lib/immich/build/geodata"
    install -Dm644 cities500.txt "${pkgdir}/usr/lib/immich/build/geodata/cities500.txt"
    install -Dm644 admin1CodesASCII.txt "${pkgdir}/usr/lib/immich/build/geodata/admin1CodesASCII.txt"
    install -Dm644 admin2Codes.txt "${pkgdir}/usr/lib/immich/build/geodata/admin2Codes.txt"
    install -Dm644 ne_10m_admin_0_countries.geojson "${pkgdir}/usr/lib/immich/build/geodata/ne_10m_admin_0_countries.geojson"
    date --iso-8601=seconds | tr -d "\n" > "${pkgdir}/usr/lib/immich/build/geodata/geodata-date.txt"

    # install systemd service files
    install -Dm644 immich-server.service "${pkgdir}/usr/lib/systemd/system/immich-server.service"
    install -Dm644 immich-machine-learning.service "${pkgdir}/usr/lib/systemd/system/immich-machine-learning.service"

    # install configuration files
    install -Dm644 immich.sysusers "${pkgdir}/usr/lib/sysusers.d/immich.conf"
    install -Dm644 immich.tmpfiles "${pkgdir}/usr/lib/tmpfiles.d/immich.conf"
    install -Dm644 immich.conf "${pkgdir}/etc/immich.conf"
    install -Dm644 nginx.immich.conf "${pkgdir}/usr/share/doc/immich/examples/nginx.conf"
    
    # install lock file (from base-images)
    # TODO this lock file is used to determine versions for ffmpeg, libheif and others
    # it will not reflect the arch installed versions but only other option is to have
    # it generated dynamically on server start - do we need to do this?
    cd "${srcdir}/base-images"
    install -Dm644 server/bin/build-lock.json "${pkgdir}/usr/lib/immich/build/build-lock.json"
}

package_immich-cli() {
    depends=('nodejs>=20')

    cd "${srcdir}/${pkgbase}-${pkgver}/cli"
    install -dm755 "${pkgdir}/usr/lib/immich/cli"
    cp -r node_modules "${pkgdir}/usr/lib/immich/cli/node_modules"
    cp -r dist "${pkgdir}/usr/lib/immich/cli/dist"
    cp -r LICENSE "${pkgdir}/usr/lib/immich/cli/LICENSE"

    # setup symlink to allow immich command to be run from shell
    chmod +x "${pkgdir}/usr/lib/immich/cli/dist/index.js"
    install -dm755 "${pkgdir}/usr/bin"
    ln -s ../lib/immich/cli/dist/index.js "${pkgdir}/usr/bin/immich"
}
