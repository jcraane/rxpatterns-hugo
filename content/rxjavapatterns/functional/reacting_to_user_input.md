---
title: "Reacting To User Input"
date: 2020-07-16T11:35:28+02:00
draft: false
---
This demo consists of a user interface containing a zipcode and housenumber fields. We want to retrieve the addess for zipcode and housenumber combination when both fields are filled-in by the user and valid. When the housenumber changes, we want to wait 300ms before exeuting the request. This demo is written in Android/Kotlin and uses an extension function to convert the EditText input to an Observable.  

```java
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val myApi = Retrofit.Builder()
                .baseUrl("SERVER_URL")
                .addCallAdapterFactory(RxJava2CallAdapterFactory.createWithScheduler(Schedulers.io()))
                .addConverterFactory(GsonConverterFactory.create(GsonBuilder().create()))
                .build().create(MyApi::class.java)

        val zipcodeSubject = zipCode.toObservable()
        val houseNumberSubject = houseNumber.toObservable()

        Observable.combineLatest<String, String, Pair<String, String>>(
                zipcodeSubject
                        .filter { postcode -> zipCodeRegExp.matcher(postcode).matches() },
                houseNumberSubject
                        .map { it.trim() }
                        .filter({ it.isNotEmpty() })
                        .debounce(300, TimeUnit.MILLISECONDS),
                BiFunction { t1, t2 -> Pair(t1, t2) }
        )
                .switchMap { myApi.retrieveAddress(it.first, it.second) }
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe({ Toast.makeText(this, it.street, Toast.LENGTH_SHORT).show() }, { it.printStackTrace() })
    }

    companion object {
        private val zipCodeRegExp = Pattern.compile("[0-9]{4}[A-z]{2}")
    }
}

fun EditText.toObservable(): Observable<String> {
    val subject = PublishSubject.create<String>()
    this.addTextChangedListener(object : TextWatcher {
        override fun afterTextChanged(p0: Editable?) {
            // Ignore
        }

        override fun beforeTextChanged(p0: CharSequence?, p1: Int, p2: Int, p3: Int) {
            // Ignore
        }

        override fun onTextChanged(charSequence: CharSequence, start: Int, before: Int, count: Int) {
            subject.onNext(charSequence.toString())
        }
    })
    return subject
}
```
