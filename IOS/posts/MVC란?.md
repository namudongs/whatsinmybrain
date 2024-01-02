## 개요

졸업준비도 마무리하고 새해도 밝았겠다, 본격적으로 취업을 위한 프로젝트를 진행해보려고 한다.
그래서 이제 새로운 프로젝트에 사용할 아키텍쳐나, 프레임워크, 라이브러리 등을 살펴보고 포스팅할 예정이다.

첫 번째로는 IOS에서 MVC구조란 무엇인지 알아볼 것이다.
MVC란 Model-View-Controller의 세 부분으로 앱을 분리하는 것을 목표로 하는 아키텍쳐 패턴이다.
이렇게 하는 이유는 코드의 재사용성을 높이고, 유지 관리를 쉽게 하며, 각 부분의 독립성을 강화해 서로간의 충돌로 인한 예기치 않은 문제들을 미연에 방지하기 위해서다.

## Model

Model은 데이터와 데이터를 처리하는 비즈니스 로직을 담당한다.
여기서 비즈니스 로직이란 앱의 핵심 기능을 구현하는 코드 부분으로, 네트워크 처리나, 데이터 파싱, 앱 내 연산 등이 있다.
또한 Model은 데이터로 사용하는 구조체도 포함하는데 데이터를 파싱해서 저장하는 다음과 같은 구조체를 의미한다.

```Swift
//WeatherKit을 사용하여 날씨 데이터를 받아와 저장하는 구조체
struct Weather {
    let location: String
    let temperature: Double
    let humidity: Double
    let windSpeed: Double
    let condition: WeatherCondition
    let forecast: [DailyForecast]

    init(from weatherData: WeatherData, at location: String) {
        self.location = location
        self.temperature = weatherData.currentWeather.temperature.value
        self.humidity = weatherData.currentWeather.humidity
        self.windSpeed = weatherData.currentWeather.wind.speed.value
        self.condition = weatherData.currentWeather.condition
        self.forecast = weatherData.forecast.daily
    }
}
```

이렇게 데이터를 저장하는 구조체 외에도, API 호출을 담당하는 WeatherService와 같은 Service 코드나,
사용자의 위치를 가져오는 LocationManager와 같은 Manager 코드도 포함될 수 있다.

Color와 같은 상수나, String의 추가적인 기능 등 Util이나 Extension 코드들도 Model에 포함된다.

## View

View 부분은 사용자에게 정보를 표시하는 인터페이스를 담당한다.
위처럼 WeatherKit을 이용해 날씨 정보를 받아왔다면, 이를 어떻게 표시할지 구성하는 코드를 포함한다.
버튼이나 레이블, 이미지 뷰, 텍스트 필드 같은 UI 요소들과 오토레이아웃 코드들도 포함된다.

View에서는 재사용성이 강조되는데, 자주 쓰이는 버튼이나 텍스트필드와 같은 코드들은 추상화해서 사용하는게 좋다.
중요한 점은 View와 Model 사이에서는 어떠한 상호작용도 이루어지면 안되고, 비즈니스 로직을 포함해서도 안된다.
이러한 상호작용은 Controller를 통해 간접적으로만 이루어져야 한다.
즉, View의 요청을 받아 Controller가 Model로 전달하고, Model의 데이터 변화는 Controller가 View에 전달해야 한다.

Storyboard로 View를 구현한다면 Main.storyboard와 같은 .storyboard 파일이 View에 해당할 것이다.

## Controller

Controller 는 Model-View 사이의 연결고리 역할을 한다.
