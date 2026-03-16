# Debian Android Build Setup

Bu rehber, `/home` altini kullanmadan Debian uzerinde Android APK uretebilmek icin gerekli kurulum adimlarini verir.

Kurulum modeli:

- Sistem paketleri `apt` ile kurulur
- Android Studio `snap` ile kurulur
- Android SDK `/opt/android-sdk` altina kurulur
- Gradle cache gecici olarak `/tmp/gradle-home` altina yonlendirilir

## Sistem Kurulumu

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk unzip wget git snapd

sudo snap install android-studio --classic

sudo mkdir -p /opt/android-sdk/cmdline-tools
cd /tmp
wget https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip -O cmdline-tools.zip
unzip -q cmdline-tools.zip
sudo mkdir -p /opt/android-sdk/cmdline-tools/latest
sudo mv cmdline-tools/* /opt/android-sdk/cmdline-tools/latest/

echo 'export ANDROID_HOME=/opt/android-sdk' | sudo tee /etc/profile.d/android-sdk.sh >/dev/null
echo 'export ANDROID_SDK_ROOT=/opt/android-sdk' | sudo tee -a /etc/profile.d/android-sdk.sh >/dev/null
echo 'export PATH=$PATH:/opt/android-sdk/cmdline-tools/latest/bin:/opt/android-sdk/platform-tools' | sudo tee -a /etc/profile.d/android-sdk.sh >/dev/null

source /etc/profile.d/android-sdk.sh

sudo env ANDROID_HOME=/opt/android-sdk ANDROID_SDK_ROOT=/opt/android-sdk /opt/android-sdk/cmdline-tools/latest/bin/sdkmanager --licenses
sudo env ANDROID_HOME=/opt/android-sdk ANDROID_SDK_ROOT=/opt/android-sdk /opt/android-sdk/cmdline-tools/latest/bin/sdkmanager \
  "platform-tools" \
  "platforms;android-35" \
  "build-tools;35.0.0"
```

## Kontrol

```bash
java -version
javac -version
```

## APK Build

```bash
cd /home/levent/projects/litter
export ANDROID_HOME=/opt/android-sdk
export ANDROID_SDK_ROOT=/opt/android-sdk
export GRADLE_USER_HOME=/tmp/gradle-home
./apps/android/gradlew -p apps/android :app:assembleRemoteOnlyDebug
```

APK tipik olarak burada olur:

```bash
apps/android/app/build/outputs/apk/remoteOnly/debug/
```

## Not

- `codex app-server` tarafinda `wss://` dinleme yoktur; `wss` TLS termination Nginx ustunden saglanir.
- Android istemci bu repo guncellemesiyle `wss://host[:port][/path][?query]` formatindaki Codex URL'lerini kabul eder.
