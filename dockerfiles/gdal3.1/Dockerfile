ARG IMAGE_VERSION

FROM ${IMAGE_VERSION}:build as builder

ENV \
  GEOS_VERSION=3.8.1 \
  PROJ_VERSION=7.1.1 \
  LIBTIFF_VERSION=4.1.0

# libtiff
RUN mkdir /tmp/libtiff \
  && curl -sfL https://download.osgeo.org/libtiff/tiff-${LIBTIFF_VERSION}.tar.gz | tar zxf - -C /tmp/libtiff --strip-components=1 \
  && cd /tmp/libtiff \
  && CFLAGS="-O2 -Wl,-S" CXXFLAGS="-O2 -Wl,-S" ./configure --prefix=$PREFIX \
  && make -j $(nproc) --silent && make install \
  && rm -rf /tmp/libtiff

# proj
RUN mkdir /tmp/proj && mkdir /tmp/proj/data \
   && curl -sfL http://download.osgeo.org/proj/proj-$PROJ_VERSION.tar.gz | tar zxf - -C /tmp/proj --strip-components=1 \
   && curl -sfL http://download.osgeo.org/proj/proj-datumgrid-latest.tar.gz | tar zxf - -C /tmp/proj/data \
   && cd /tmp/proj \
   && LDFLAGS="-Wl,-rpath,'\$\$ORIGIN'" CFLAGS="-O2 -Wl,-S" CXXFLAGS="-O2 -Wl,-S" ./configure --prefix=$PREFIX --disable-static --enable-lto \
   && make -j $(nproc) --silent && make install \
   && rm -rf /tmp/proj

# geos
RUN mkdir /tmp/geos \
  && curl -sfL http://download.osgeo.org/geos/geos-$GEOS_VERSION.tar.bz2 | tar jxf - -C /tmp/geos --strip-components=1 \
  && cd /tmp/geos \
  && CFLAGS="-O2 -Wl,-S" CXXFLAGS="-O2 -Wl,-S" ./configure --prefix=$PREFIX --disable-static \
  && make -j $(nproc) --silent && make install \
  && rm -rf /tmp/geos

ENV PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig/

# gdal
# Use latest commits from Github instead of release
# "release/3.1" is the up to date 3.1.4 github branch
RUN mkdir /tmp/gdal \
  && curl -sfL https://github.com/OSGeo/gdal/archive/release/3.1.tar.gz | tar zxf - -C /tmp/gdal --strip-components=2

RUN cd /tmp/gdal \
  && touch config.rpath \
  && LIBXML2_CFLAGS=$(xml2-config --cflags) LIBXML2_LIBS=$(xml2-config --libs) LDFLAGS="-Wl,-rpath,'\$\$ORIGIN'" CFLAGS="-O2 -Wl,-S" CXXFLAGS="-O2 -Wl,-S" ./configure \
      --disable-debug \
      --enable-lto \
      --prefix=$PREFIX \
      --with-crypto \
      --with-curl \
      --with-expat \
      --with-hdf4=$PREFIX \
      --with-hdf5=$PREFIX \
      --with-libtiff=$PREFIX \
      --with-hide-internal-symbols=yes \
      --with-geos=$PREFIX/bin/geos-config \
      --with-geotiff=internal \
      --with-rename-internal-libgeotiff-symbols \
      --with-jpeg=$PREFIX \
      --with-rename-internal-libtiff-symbols \
      --with-netcdf=$PREFIX \
      --with-openjpeg \
      --with-pg=yes \
      --with-png \
      --with-proj=$PREFIX \
      --with-sqlite3=$PREFIX \
      --with-xml2=yes \
      --with-webp=$PREFIX \
      --with-zstd=$PREFIX \
      --without-cfitsio \
      --without-ecw \
      --without-fme \
      --without-freexl \
      --without-jpeg12 \
      --without-gif \
      --without-gnm \
      --without-lerc \
      --without-libtool \
      --without-pcraster \
      --without-pcidsk \
      --disable-all-optional-drivers \
      --enable-driver-pds \
      --enable-driver-shape \
      --enable-driver-gpkg \
      --enable-driver-sentinel2 \
      --enable-driver-xyz \
      --enable-driver-flatgeobuf \
      --enable-driver-mvt \
      --enable-driver-wmts \
      --enable-driver-wms \
      --enable-driver-mbtiles \
      --enable-driver-gpkg \
      --enable-driver-gml \
      --enable-driver-pgdump

RUN cd /tmp/gdal \
  && make -j $(nproc) --silent && make install \
  && rm -rf /tmp/gdal

# from https://github.com/pypa/manylinux/blob/d8ef5d47433ba771fa4403fd48f352c586e06e43/docker/build_scripts/build.sh#L133-L138
# Install patchelf (latest with unreleased bug fixes)
ENV PATCHELF_VERSION 0.10
RUN mkdir /tmp/patchelf \
  && curl -sfL https://github.com/NixOS/patchelf/archive/$PATCHELF_VERSION.tar.gz | tar zxf - -C /tmp/patchelf --strip-components=1 \
  && cd /tmp/patchelf && ./bootstrap.sh && ./configure \
  && make -j $(nproc) --silent && make install \
  && cd / && rm -rf /tmp/patchelf

# Move /lib64 (libjpeg) to /lib
RUN mv $PREFIX/lib64/lib* $PREFIX/lib/

# FIX
RUN for i in $PREFIX/bin/*; do  patchelf --force-rpath --set-rpath '$ORIGIN/../lib' $i; done

# Build final image
FROM lambci/lambda-${IMAGE_VERSION}:build  as runner

ENV PREFIX /opt
COPY --from=builder /opt/lib/ $PREFIX/lib/
COPY --from=builder /opt/include/ $PREFIX/include/
COPY --from=builder /opt/share/ $PREFIX/share/
COPY --from=builder /opt/bin/ $PREFIX/bin/

ENV \
  GDAL_VERSION=3.1.4 \
  GDAL_DATA=$PREFIX/share/gdal \
  PROJ_LIB=$PREFIX/share/proj \
  GDAL_CONFIG=$PREFIX/bin/gdal-config \
  GEOS_CONFIG=$PREFIX/bin/geos-config \
  PATH=$PREFIX/bin:$PATH
