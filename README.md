

# 키워드

- RecyclerView
- ViewPager2
- Firebase Remote Config

# 구현 목록

- 코드 수정 없이 명언 추가
- 코드 수정 없이 이름 숨김
- 무한 스와이프

# 개발 과정

## 1. 기본 UI 구성하기

### 레이아웃 구현

이번 어플은 UI가 단순하다. 말 그대로 명언을 보여주는 어플이기에 대사?와 그 사람의 이름?만 있으면 된다. 따라서 두 개의 `TextView` 로 구현했다. 우선 옆으로 스와이핑이 가능한 `ViewPage` 와 `ViewPage` 내부에 들어가는 각각의 페이지들을 별도로 분리했다. 내부에 들어가는 페이지는 동적으로 변화해야 하므로 `RecyclerView` 로 구현했다. 우선 내부에 들어가는 `RecyclerView` 의 레이아웃이다. 

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/quoteTextView"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginHorizontal="40dp"
        android:ellipsize="end"
        android:gravity="end|center_vertical"
        android:maxLines="6"
        android:textSize="30sp"
        app:flow_verticalBias="0.4"
        app:layout_constraintBottom_toTopOf="@+id/nameTextView"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_chainStyle="packed" />

    <TextView
        android:id="@+id/nameTextView"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="15dp"
        android:ellipsize="end"
        android:gravity="end|center_vertical"
        android:maxLines="1"
        android:textSize="20sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="@+id/quoteTextView"
        app:layout_constraintStart_toStartOf="@+id/quoteTextView"
        app:layout_constraintTop_toBottomOf="@+id/quoteTextView" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

이 `.xml` 와 연결할 Activity를 구현하자. 일단 공식 문서를 살펴보자. 

[안드로이드 공식 문서 - RecyclerView](https://developer.android.com/guide/topics/ui/layout/recyclerview?hl=ko)

![](https://images.velog.io/images/k906506/post/3658c18d-0eaf-4cdc-9a8e-51a44ae3fbdc/image.png)

오늘 구현할 어플처럼 View가 동적으로 변해야하는 경우에 `RecyclerView` 를 사용한다. `RecyclerView` 를 사용하기 위해선 몇 가지 작업을 실시해야 한다.

1. 목록 또는 그리드의 모양을 결정
2. 목록에 표시할 각 요소의 모양과 동작 방식을 설계
3. 이를 연결하는 `Adapter` 를 정의

1번의 경우 아래 코드처럼 `FrameLayout` 으로 구현해줬다.

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/viewPager"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <ProgressBar
        android:id="@+id/progressBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center" />

</FrameLayout>
```

2번의 경우 위에서 두 개의 `TextView` 로 정의했다. 목록과 각 요소의 모양을 구현했으니 이제 동작 방식을 설계해야한다

### 동작 방식 설계

세 가지의 메소드를 재정의해야한다. `onCreateViewHolder()` , `onBindViewHolder()` , `getItemCount()` 가 있다. 우선 가장 쉬운 것부터 살펴보자. 

```kotlin
override fun getItemCount() = Int.MAX_VALUE
```

`getItemCount()` 의 경우 `RecycleView` 에서 보여줄 데이터의 크기를 가져온다. 예를 들어 주소록 앱에서는 총 주소의 개수가 해당한다. 이번 앱에서는 명언의 개수가 해당될 것이다. 하지만 `List<Quote>.size` 가 아니라 `MAX.VALUE` 로 설정한 것을 볼 수 있다. 이는 무한 스크롤을 구현하기 위해서인데 리스트의 크기로 설정하면 딱 리스트의 목록만 보여주게 되므로 유한 스크롤로 구현이 된다. 가령 5개의 목록이 있는 경우 0 ~ 4 index를 갖게 된다. 이를 무한 스크롤로 구현하기 위해서 아이템의 개수를 나름 무한한? 21억으로 맞춰줬다.  

```kotlin
	override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        QuoteViewHolder(
            LayoutInflater.from(parent.context)
                .inflate(R.layout.item_quote, parent, false)
        )
```

다음은 `onCreateviewHolder` 이다. 말 그대로 `ViewHolder` 를 새로 만들때 이 메소드를 호출한다. `ViewHolder` 와 `View` 를 생성하고 초기화 하지만 뷰 내부의 콘텐츠는 채우지 않는다. 그 이유는 아직 binding 되지 않았기 때문이다. 별도의 클래스 `QuoteViewHolder` 를 선언하고 할당해줬는데  이 클래스에는 어떻게 보여줄지가 정의되어 있다. 

```kotlin
class QuoteViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        private val quoteTextView: TextView = itemView.findViewById(R.id.quoteTextView)
        private val nameTextView: TextView = itemView.findViewById(R.id.nameTextView)

        fun bind(quote: Quote, isNameRevealed: Boolean) {
            quoteTextView.text = "\"${quote.quote}\""
            if (isNameRevealed) {
                nameTextView.text = "- ${quote.name}"
                nameTextView.visibility = View.VISIBLE
            } else {
                nameTextView.visibility = View.GONE
            }
        }
    }
```

우선 보여줄 View를 인자로 받는다. 이후 `findViewById` 를 사용해서 레이아웃과 연결해준다. 

```jsx
	override fun onBindViewHolder(holder: QuoteViewHolder, position: Int) {
        val actualPosition = position % quotes.size
        holder.bind(quotes[actualPosition], isNameRevealed)
    }
```

`onBindViewHolder` 의 경우 말 그대로 데이터와 연결해주는 메소드이다. `viewHolder` 의 bind 메소드를 호출하고 값들을 넘겨준다.  마지막은 `adapter` 와 연결해주는 것이다. `MainActivity` 에서 이를 연결해주면 된다.

```jsx
	private fun displayQuotesPager(quotes: List<Quote>, isNamedRevealed: Boolean) {
        val adapter = QuotesPageAdapter(
            quotes,
            isNamedRevealed
        )
        viewPager.adapter = adapter
        viewPager.setCurrentItem(adapter.itemCount / 2, false)
    }
```

## 2. 페이지 전환

좌우로 슬라이딩을 할 때 페이지 전환의 효과를 넣어주고 싶었다. 이 때 `setPageTransfer` 를 호출해서 사용했다. 

```jsx
	private fun initViews() {
        viewPager.setPageTransformer { page, position ->
            when {
                position.absoluteValue >= 1.0F -> {
                    page.alpha = 0F
                }
                position == 0F -> {
                    page.alpha = 1f
                }
                else -> {
                    page.alpha = 1F - 2 * position.absoluteValue
                }
            }
        }
    }
```

현재 보고 있는 페이지를 중심으로 좌우로 +1 -1 ... 이렇게 값을 갖게 되는데 이를 이용해서 넘어가면서 흐려지게끔 구현했다. 이 때 `alpha` 값을 변경해줬다. 공식 문서에 더 많은 기능들이 나와있다.

[안드로이드 공식 문서 - ViewPager2](https://developer.android.com/training/animation/screen-slide?hl=ko)

앱을 실행해보자. 

![](https://images.velog.io/images/k906506/post/2f27fe82-f577-4432-9175-212647434da5/bandicam%202021-09-26%2021-38-20-009.jpg)

처음에 `Firebase` 와 연결해야하므로 이 때 `ProgressBar` 를 이용해서 이를 표시해줬다. 

![](https://images.velog.io/images/k906506/post/91773224-e14c-4160-9999-0936a02dd24f/image.png)

좌우로 넘길 때 흐려지는 것을 볼 수 있다. `alpha` 로 설정해 준 것이다. 

한번 `Remote Config` 로 값을 변경해보자.

![](https://images.velog.io/images/k906506/post/8eb4de9f-5d88-4f07-83bf-950da7b48ec1/image.png)

이름이 보이지 않도록 설정하고 앱을 재실행해보면 이름이 보이지 않는 것을 볼 수 있다. remote config를 사용하면 앱의 빌드 없이도 앱의 정보를 변경할 수 있다는 장점이 있다.

![](https://images.velog.io/images/k906506/post/e4f28a1c-1114-49ef-ae43-60a98df97e77/bandicam%202021-09-26%2021-41-19-572.jpg)

# 느낀 점

ㅈㄴ어렵다... 플러터가 확실히 쉬운 것 같다. 열심히 분발해야겠다....
