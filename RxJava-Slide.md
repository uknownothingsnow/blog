Observerable asynchronously data stream
lowlevel threading
synchronize
thread safety
conconrrent data structure

why FRP
concurrent programming is difficult
transform compose asynchronously data
high-level abstraction
standard error handling

Push VS Pull
for(Iterator<Tweet> tweet = timeline.iterator(); iterator.hasNext();) {

}

event Iterable(pull) Observable(push)
retrieveData T next()  onNext(T)
discover error throw Exception onError(Exception)
complete !hasNext()   onComplete()

everything is stream

The Observable data type can be thought of as a "push" equivalent to Iterable which is "pull". With an Iterable, the consumer pulls values from the producer and the thread blocks until those values arrive. By contrast with the Observable type, the producer pushes values to the consumer whenever values are available. This approach is more flexible, because values can arrive synchronously or asynchronously.

The Observable type adds two missing semantics to the Gang of Four's Observer pattern, which are available in the Iterable type:

The ability for the producer to signal to the consumer that there is no more data available.
The ability for the producer to signal to the consumer that an error has occurred.



http://f2prateek.com/2015/10/05/rx-preferences/

Callers must always know the preference key and type.
No support for storing custom types out of the box.
Callers cannot listen for changes to individual keys.

For example, RxPreferences and RxBinding can be combined to hand roll your own simplified CheckBoxPreference.

@Inject @LocationPreference BooleanPreference locationPreference;
@BindView(R.id.check_box) CheckBox checkBox;

// Update the checkbox when the preference changes.
locationPreference.asObservable()
  .observeOn(AndroidSchedulers.mainThread())
  .subscribe(RxCompoundButton.checked(checkBox));

// Update preference when the checkbox state changes.
RxCompoundButton.checkedChanges(checkBox)
  .skip(1) // Skip the initial value.
  .subscribe(locationPreference.asAction());
