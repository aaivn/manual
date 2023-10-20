# Драйвера NVIDIA в Fedora
Установка выполняется по разделам статьи: [Howto/NVIDIA](https://rpmfusion.org/Howto/NVIDIA#Current_GeForce.2FQuadro.2FTesla)

* Fedora 38
* карта `NVIDIA GeForce GTX 950`

## Installing the drivers
### Current GeForce/Quadro/Tesla
```
# 1. предварительное обновление всех пакетов
$ sudo dnf update -y

# 2. перезагрузка системы
# 3. установка драйверов
$ sudo dnf install akmod-nvidia
$ sudo dnf install xorg-x11-drv-nvidia-cuda

# 4. ожидание завершения
$ modinfo -F version nvidia
```

### KMS
KMS означает «Настройка режима ядра», что является противоположностью «Настройка режима пользователя». Эта функция позволяет установить разрешение экрана на стороне ядра один раз (при загрузке), а не после входа в систему из диспетчера дисплея.

* **включить**
    * `sudo grubby --update-kernel=ALL --remove-args='nvidia-drm.modeset=1'`
* отключить (если нужно)
    * `sudo grubby --update-kernel=ALL --args='nvidia-drm.modeset=1'`


### Suspend
```
# 1. установка пакета питания
$ sudo dnf install xorg-x11-drv-nvidia-power

# 2. включение службы
$ sudo systemctl enable nvidia-{suspend,resume,hibernate}
```

### Vulkan
Не устанавливается, есть `mesa-vulkan-drivers`
```
$ sudo dnf  list --installed | grep vulkan
mesa-vulkan-drivers.x86_64                           23.1.8-1.fc38                       @updates                        
vulkan-loader.x86_64                                 1.3.243.0-1.fc38                    @updates
```

### Wayland
На экране логина в настройках переключаем на режим GNOME Xrg, чтобы работало видеоускорение.

### NVENC/NVDEC
Пакет `xorg-x11-drv-nvidia-cuda-libs` уже устанавливался выше

### nvidia-xconfig
По инструкции с офф сайта [set-nvidia-as-primary-gpu](https://docs.fedoraproject.org/en-US/quick-docs/set-nvidia-as-primary-gpu-on-optimus-based-laptops/#_step_8_edit_the_x11_configuration)

```
# 1. установка пакета `xrandr`
$ sudo dnf install xrandr

# 2. копирование конфига
sudo cp -p /usr/share/X11/xorg.conf.d/nvidia.conf /etc/X11/xorg.conf.d/nvidia.conf

# 3. включение карты nvidia основной. добавить параметр в скопированный конфиг
Option "PrimaryGPU" "yes"

# 4. Проверяем, что карта определяется
$ glxinfo | grep -E "OpenGL vendor|OpenGL renderer"

OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: NVIDIA GeForce GTX 950/PCIe/SSE2
```

### VDPAU/VAAPI
Включение видеоускорения
```
# 1. установка драйверов
$ sudo dnf install nvidia-vaapi-driver libva-utils vdpauinfo

# 2. проверка, что следующая команда отрабатывает и не вызывает ошибки
$ vdpauinfo

# 3. проверка, что следующая команда отрабатывает и не вызывает ошибки
$ vainfo
# у меня тут ошибка, по-поэтому смотрим инструкцию по конфигурации ниже
libva error: vaGetDriverNameByIndex() failed with unknown libva error, driver_name = (null)
```

[Configuring_VA-API](https://wiki.archlinux.org/title/Hardware_video_acceleration#Configuring_VA-API)

```
# по логам смотрим какой драйвер используется
$ journalctl -b | grep -iE 'vdpau | dri driver'
# или
$ grep -iE 'vdpau | dri driver' ~/.local/share/xorg/Xorg.0.log

[    19.144] (II) modeset(G0): [DRI2]   DRI driver: iris
[    19.144] (II) modeset(G0): [DRI2]   VDPAU driver: va_gl
```

Используется не подходящий драйвер. Пробуем переопределить используемый драйвер переменной окружения `LIBVA_DRIVER_NAME`.

```
# создаем файл профиля для nvidia
$ sudo $EDITOR /etc/profile.d/nvidia.sh

# перечитываем 
$ source /etc/profile.d/nvidia.sh

# проверяем, что указана
$ echo $LIBVA_DRIVER_NAME

# и проверяем повторно
$ vainfo
Trying display: wayland
Trying display: x11
libva info: VA-API version 1.18.0
libva info: User environment variable requested driver 'nvidia'
libva info: Trying to open /usr/lib64/dri/nvidia_drv_video.so
libva info: Found init function __vaDriverInit_1_0
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.18 (libva 2.18.2)
vainfo: Driver version: VA-API NVDEC driver [egl backend]
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            :	VAEntrypointVLD
      VAProfileMPEG2Main              :	VAEntrypointVLD
      VAProfileVC1Simple              :	VAEntrypointVLD
      VAProfileVC1Main                :	VAEntrypointVLD
      VAProfileVC1Advanced            :	VAEntrypointVLD
      VAProfileH264Main               :	VAEntrypointVLD
      VAProfileH264High               :	VAEntrypointVLD
      VAProfileH264ConstrainedBaseline:	VAEntrypointVLD
      VAProfileHEVCMain               :	VAEntrypointVLD
      VAProfileVP8Version0_3          :	VAEntrypointVLD
      VAProfileVP9Profile0            :	VAEntrypointVLD
      VAProfileHEVCMain10             :	VAEntrypointVLD
```

