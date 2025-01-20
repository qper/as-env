# as-env

''' find /usr/bin /usr/local/bin /usr/sbin /sbin /bin /lib /lib64 /usr/lib /usr/lib64 /opt /usr/libexec \
  -type f \( -executable -o -name "*.so*" -o -name "*.a" \) -print0 | sort -z | xargs -0 sha256sum'''
