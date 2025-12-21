```python
sudo dnf install -y policycoreutils-python-utils
sudo semanage fcontext -a -t bin_t '/var/lib/kubelet/plugins/ru\.yandex\.s3\.csi/geesefs' \
  || sudo semanage fcontext -m -t bin_t '/var/lib/kubelet/plugins/ru\.yandex\.s3\.csi/geesefs'
sudo restorecon -vF /var/lib/kubelet/plugins/ru.yandex.s3.csi/geesefs
ls -Z /var/lib/kubelet/plugins/ru.yandex.s3.csi/geesefs

```
