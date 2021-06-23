## Profiling GPU Rendering

안드로이드에는 UI 를 필요이상으로 렌더링하거나 스레드와 GPU 연산을 오래 실행하는 앱에서 발생할 수 있는 이슈를 시각화하여 렌더링과 최적화를 도울수 있는 개발자 툴이 있다. 프로파일을 Android 4.1(API 16 ) 이상에서 사용 가능하며 미리 해당 기능을 활성화하는 작업이 필요하다. 다음과 같은 순서로 진행하면 활성화 할 수 있다.

 1. 설정에서 개발자 옵션을 선택
 2. 모니터링 섹션에서 Profile GPU Rendering 또는 Profile HWUI rendering 선택
 3. 화면에 막대로 표시를 선택

Profile GPU Rendering 활성화되면 아래와 같이 표시된다.

![](https://cdn-images-1.medium.com/max/2000/1*pMOgwlRuRGRubD6y5PKrlQ.png)

* 가로축에 각 세로 막대는 하나의 프레임을 나타내며, 각각의 세로 막대의 높이는 하나의 프레임을 렌더링하는데 걸린 시간을 나타낸다.

* 중간에 가로로 된 녹색 가로선은 16ms(milliseconds) 을 나타내며, 초당 60 프레임을 지원하려면 세로 막대는 녹색 가로선 아래에 유지되도록 해야한다. 이 녹색 가로선을 초과하는 경우에는 애니메이션이 일시 중단될 수도 있다. 이 외 노란 가로선은 21ms (47 fps) 빨간 가로선은 31ms (32 fps) 을 나타낸다.

![](https://cdn-images-1.medium.com/max/2000/0*rJMnRq6DAji2Xnul.png)

세로 막대 차트는 위와 같이 8개의 단계로 표시된다.

* **프로세스/스왑버퍼(Swap)**: 안드로이드는 디스플레이 목록을 GPU 에 제출하는것을 끝내면, 시스템은 그래픽 드라이버에 현재 프레임이 완료되었음을 알리며 드라이버를 업데이트된 이미지를 화면에 표시한다. CPU 와 GPU 는 병렬로 처리되는데 CPU 가 GPU 처리보다 빠른 경우, 프로세스간 통신 큐가 가득찰 수 있다. CPU 는 큐에 공간이 생길때까지 기다린다. 큐가 가득한 상태는 스왑버퍼 상태에서 자주 발생하는데 이 단계에서 프레임 전체의 명령어가 제출되기 때문이다. 명령어 실행 단계와 비슷하게 GPU 에서 일어나는 일의 복잡성을 줄여 문제를 해결할 수 있다.

* **명령어 실행(Issue)**: 디스플레이 목록을 화면에 그리기 위한 모든 명령어를 실행하기 위해 걸리는 시간을 측정한다. 주어진 프레임에서 렌더링하는 디스플레이 목록과 수량을 직접적으로 측정한다. 내재적으로 많은 그리기 동작이 있는 경우, 오랜 시간이 걸릴수 있다.

* **동기화/업로드(Upload)**: 현재 프레임에서 비트맵 객체가 CPU 메모리에서 GPU 메모리로 전송되는데 걸리는 시간을 측정한다. 오랜 시간이 걸리는 경우는 작은 리소스가 많이 로드되어 있거나 적지만 큰 리소스가 로드되어 있는 경우이다. 비트맵 해상도가 실제 디스플레이 해상도보다 큰 사이즈로 로드되어 있지 않도록 하거나 동기화 전에 비동기로 미리 업로드해서 실행 시간을 줄일 수 있다.

* **그리기(Draw)**: 백그라운드 혹은 텍스트 그리기와 같은 뷰를 그리기 명령어로 번역하는 단계로 시스템은 명령어를 디스플레이 목록에 캡처한다. 무효화된 뷰에서 onDraw 호출을 실행해 걸리는 모든 시간을 측정하며, 오랜 시간이 걸린다는 것은 무효화된 뷰가 많다는 것을 의미한다. 뷰가 무효화되는 경우 뷰의 디스프레이 목록을 재생성한다. 혹은 커스텀한 뷰가 onDraw 할때 복잡한 로직을 가질 때도 실행시간이 오랜 걸린다.

* **측정 및 배치(Measure)**: 안드로이드에서 뷰를 스크린에 그리기 위해 측정하고 배치하는데 걸리는 시간을 측정한다. 일반적으로 오랜 시간이 걸리는 경우는 배치할 뷰가 너무 많거나 혹은 계층구조의 잘못된 장소에서 중복 계산이 이뤄지며, Traceview와 Systrace 를 사용해서 호출 스택을 확인하여 문제를 파악할 수 있다.

* **애니메이션(Anim)**: 현재 프레임에서 실행되는 애니메이션에 대해 걸리는 시간을 측정한다. 일반적으로 오랜 시간이 걸리는 경우는 애니메이션의 속성 변경으로 인해 발생한다.

* **입력 처리(Input)**: 앱이 입력 이벤트를 처리하는 시간으로 입력 이벤트 콜백의 결과로 호출된 코드를 실행하는데 걸리는 시간을 나타낸다. 일반적으로 오랜 시간이 걸리는 경우는 메인쓰레드에서 발생하며, 작업을 최적화하거나 다른 쓰레드를 이용해서 작업을 실행하도록 한다.

* **기타(Misc)**: 렌더링 시스템이 작업을 수행하는 시간외에 렌더링과 관련 없는 추가적인 작업이 있다. 일반적으로 렌더링의 연속된 두 프레임 사이에서 UI 스레드에서 발생할 수 있는 일을 나타낸다. 오랜 시간이 걸리는 경우, 다른스레드에서 실행해야할 콜백, 인텐트 또는 기타 다른 작업이 있을 수 있다. Method tracing 또는 Systrace 를 사용하면 메인스레드에서 실행중인 작업을 확인해 문제를 해결할 수 있다.



아래에는 이미지 뷰에 클릭 이벤트를 설정하고 이미지 리소스를 변경하는 로직이다.
좌측에는 단순하게 이미지 리소스를 변경하는 로직만 추가하였고, 우측에는 리소스를 변경하는 과정에서 스레드를 일시 중단하는 코드를 추가하였다. 그 결과 입력처리 시간에 차이가 발생한다는 것을 확인할 수 있다.

![](https://cdn-images-1.medium.com/max/2912/1*Fq7VKzUg_c7WCcPy4ILJDw.png)



아래에는 이미지 뷰에 각각 다른 크기의 이미지 리소스를 배치하였다.
좌측에는 8.1MB 크기의 이미지 리소스를 배치하고 우측에는 16KB 크기의 이미지 리소스를 배치하였다. 그 결과 이미지를 업로드 시간에 차이가 발생한 다는 것을 확인할 수 있다.

![](https://cdn-images-1.medium.com/max/2932/1*_Cd42DYLhjYBcmPjOZroGw.png)





## GPU 오버드로우 시각화

다른 개발자 옵션으로 UI 에 컬러를 지정함으로써 오버드로우를 식별할 수 있다. 같은 프레임내에서 같은 픽셀을 한번 이상 그릴 때 오버드로우가 발생한다. 앱에서 필요 이상으로 많은 렌더링이 발생하는 곳을 시각화로 보여주며, 사용자에게 보여지지 않는 픽셀을 렌더링하기 위해 추가 GPU 작업으로 성능 문제가 발생한 수 있음을 알 수 있다. 다음과 같이 설정하면 오버드로우 시각화를 확인 할 수 있다.

 1. 설정에서 개발자 옵션을 선택
 2. 하드웨어 가속 렌더링 섹션에서 Debug GPU Overdraw 선택
 3. 화면에 오버드로 영역 표시를 선택

* **True color:** 오버드로우 없음

* **Blue color:** 오버드로우 1회

* **Green color:** 오버드로우 2회

* **Pink color:** 오버드로우 3회

* **Red color:** 오버드로우 4회 이상

뷰의 배치에 따라 각각 오버드로우가 발생한 픽셀을 확인할 수 있다.

![](https://cdn-images-1.medium.com/max/2000/1*FqRNnuyKanJHUKr7edRjwA.png)







[https://developer.android.com/topic/performance/rendering/inspect-gpu-rendering#debug_overdraw](https://developer.android.com/topic/performance/rendering/inspect-gpu-rendering#debug_overdraw)

[https://developer.android.com/topic/performance/rendering/profile-gpu](https://developer.android.com/topic/performance/rendering/profile-gpu)

