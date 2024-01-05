MacOS Sonoma가 나오고, RC버전까지 나온 상태에서 얼마 안 있으면 정식 버전이 나올 것 같아 베타로 올려봤다.

이때부터가 고난의 시작이었는데...

1\. 베타로 Sonoma를 올리면 기존 Xcode 14는 사용하지 못한다.

2\. 그래서 [Xcode 15 RC](https://developer.apple.com/download/all/?q=Xcode)를 설치해야 한다.   
3\. 설치 후 플러터 프로젝트를 실행하면 에러가 발생한다.

4\. 에러 하나를 해결하면 다른 에러가 또 생긴다.

대충 이런 상황이었다..

일단 1, 2는 그냥 설치만 하면 되는거니 쉽게 끝난다.

플러터에서 계속 에러가 나는게 문제인데..

일단 처음 발생한 에러는 이거였다.

### Error (Xcode): DT\_TOOLCHAIN\_DIR cannot be used to evaluate LIBRARY\_SEARCH\_PATHS, use TOOLCHAIN\_DIR instead

아무래도 Xcode 15 beta5 부터 발생한 에러인 것 같다. 한국어로 된 정보는 많이 없는 것 같은데..

Cocoapods 관련 문제인 듯. 깃헙부터 애플 디벨로퍼 사이트까지 뒤져본 결과 해결할 수 있었다..

플러터 프로젝트 폴더 안의 ios/Podfile 코드를 살펴보면 마지막쯤에 이런 코드가 있다.

```
post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
  end
end
```

이 코드를 아래 코드로 바꿔주면 해결된다.

```
post_install do |installer|
  installer.aggregate_targets.each do |target|
    target.xcconfigs.each do |variant, xcconfig|
      xcconfig_path = target.client_root + target.xcconfig_relative_path(variant)
      IO.write(xcconfig_path, IO.read(xcconfig_path).gsub("DT_TOOLCHAIN_DIR", "TOOLCHAIN_DIR"))
    end
  end
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      if config.base_configuration_reference.is_a? Xcodeproj::Project::Object::PBXFileReference
        xcconfig_path = config.base_configuration_reference.real_path
        IO.write(xcconfig_path, IO.read(xcconfig_path).gsub("DT_TOOLCHAIN_DIR", "TOOLCHAIN_DIR"))
      end
    end
  end
end
```

### No such module 'Flutter'

그럼 이제 이 에러가 발생한다. Flutter 모듈을 찾을 수 없다는건데.. 상세 내용을 보면 shared\_preferences 얘가 갑자기 튀어나와서 얘가 문제인가? 생각할 수 있다.

근데 얘랑은 관련 없는 것 같다.

아무리 구글링을 해보고 여러 방법들을 시도해봐도 해결이 안된다.

근데 아래 코드대로 해보니 에러 메세지가 바뀌긴 했다.

```
cd ios
rm Podfile.lock
rm Podfile
rm -rf Pods

pod cache clean --all

cd .. (상위 폴더로 되돌아가기)
flutter clean
flutter pub get

cd ios
pod install
```

[여기서](https://velog.io/@ayb226/Flutter-%EC%98%A4%EB%A5%98-%EB%AA%A8%EC%9D%8C-%EC%97%AC%EB%9F%AC%EA%B0%80%EC%A7%80-%EC%9D%B4%EC%9C%A0%EB%A1%9C-%EB%B9%8C%EB%93%9C%EA%B0%80-%EC%95%88-%EB%90%A0-%EB%95%8C-%ED%95%B4%EA%B2%B0%EB%B2%95) 찾은 방법이다. 자, 이대로 하면 다음과 같은 에러가 또 발생한다.

### Lexical or Preprocessor Issue (Xcode): 'Flutter/Flutter.h' file not found

슬슬 짜증이 났다. 왜냐하면 이 때는 빌드가 잘 되다가 에러가 떴기 때문이다..

이제까지는 빌드하고 한 10초정도면 에러 뿜뿜하면서 멈췄는데 1분 가까이 빌드하길래 이제 성공하나? 했는데..

이제는 다른 에러가 나를 맞이했다.

여기서 한 1시간가량을 찾아봤던 것 같다.

근데 내가 처음 DT\_TOOLCHAIN\_DIR 에러의 해결 방법을 찾는 애플 디벨로퍼 사이트의 스레드에서 답을 찾았다 !!

![[Pasted image 20240103153413.png|700]]

코멘트가 무려 저번주에 달렸다..

너무나도 감사하게도.. ios/Podfile의 post\_install 코드를 위에 써놓은 기존 코드가 아니라 이 코드로 바꾸니 해결됐다😇

```
post_install do |installer|
  installer.pods_project.targets.each do |target|
      target.build_configurations.each do |config|
      xcconfig_path = config.base_configuration_reference.real_path
      xcconfig = File.read(xcconfig_path)
      xcconfig_mod = xcconfig.gsub(/DT_TOOLCHAIN_DIR/, "TOOLCHAIN_DIR")
      File.open(xcconfig_path, "w") { |file| file << xcconfig_mod }
      end
  end
  installer.pods_project.targets.each do |target|
      flutter_additional_ios_build_settings(target)
      target.build_configurations.each do |config|
        config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.0'
      
      end
  end
end
```

Target은 12.0 이상으로만 설정해주면 된다. 참고로 pod install 시에 \[!\] 에러가 발생하고 빌드가 되지 않는다면,

```
# Uncomment this line to define a global platform for your project
platform :ios, '12.0' (주석처리 되어있다면 해제)
```

ios/Podfile 에서 맨 윗줄 platform이 #으로 주석처리 되어있다면 주석을 해제하고 다시 실행하면 된다😎

추가로 PhaseScriptExecution failed with a nonzero exit code 오류가 발생한다면 아래 더보기를 눌러주세요!

더보기

먼저 Xcode로 Runner를 실행해서 iOS Target을 먼저 확인해주세요.

확실하진 않지만 위의 platform만 12로 설정해 두고, 기존에 Target Version은 건들지 않는게 좋습니다!

저같은 경우는 이 오류가 Target Version도 같이 12로 바꿔서 발생했습니다.

![[Pasted image 20240103153438.png|600]]

위의 Minimum Deployments 또한 저처럼 11.0으로 이미 설정되어있다면 건들지 않는게 좋습니다.  
가장 좋은 방법은 현재 프로젝트 폴더를 지우고 맥 업데이트 이전 잘 돌아가는 코드를 다시 클론해서 만든 다음, 위에 안내드린대로  
  
1\. ios/Podfile의 post\_install 변경  
2\. 터미널로 프로젝트 폴더로 이동 후, 아래 코드 실행

```
cd ios
# Podfile.lock 삭제해도 되고, 삭제하지 않아도 됨
pod deintegrate
pod install
cd ..
flutter clean
flutter pub get
flutter run (-v)
```

위 단계만 해보시는게 좋습니다. 여러번 pod install 을 반복하다 보면 pubspec이 꼬여서 되던 것도 안되더라구요.

---

이제 Sonoma로 올린 내 맥에서도 Xcode 15와 함께 프로젝트가 잘 실행된다.

한 3시간가량을 꼬박 뒤졌던 것 같다. 그래도 덕분에 시뮬레이터도 15 Pro로 동작하고 iOS도 17.0이다.

시간은 정말 많이 썼지만 나름 좋은 경험을 했다고 생각한다.