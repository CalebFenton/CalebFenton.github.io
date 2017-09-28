title: "Filtering Popular Code and Effects on Model Accuracy"
tags:
  - android
  - research
  - machine learning
comments: true
date: 2017-09-25 00:13:37
---

I've been conducting experiments to try and improve accuracy of a malware detection tool called [Judge](http://judge.rednaga.io/). This post is about an experiment involving finding which classes and methods occur frequently in the training data and excluding them from training. The intuition is that by filtering out popular code, the features will be more representative of what's unique to each sample.

This idea, as well as several other such as model stacking and blending which I'll write about later, as well as some implementation ideas were given to me by Nikita Buchka ([@advegoc](https://twitter.com/advegoc) and [blogs](https://securelist.com/all/?author=551&filter=)).
<!-- more -->

## Fingerprinting Code

The first step in filtering out popular code is creating a fingerprinting mechanism which can produce a hash for code at the class and method level. Identical or _very_ nearly identical Java code should map to the same hash value, and any small change in the code should lead to a totally different hash.

How can you do this? You might think you can just hash the actual Dalvik executable bytes of a method to get a fingerprint, but the exact same Java code may look totally different at the byte level because of differing register layouts and virtual table lookup indices.

My approach was to transform class and method properties as well as opcodes into a string. Properties include annotation size, interface count, access flags, register count, and so on. This string representation should be highly brittle and sensitive to almost any changes. Finally, the string is hashed into a `long` using an implementation of [MurmurHash3](https://github.com/yonik/java_util/blob/master/src/util/hash/MurmurHash3.java).

The source code is available here: [DexFileFingerprinter.java](https://gist.github.com/CalebFenton/38dc21d6619d51c7609b879752f2d24e). Also note, there's a `boolean strict` flag which is true by default. This was an untested idea of building less strict fingerprint strings of methods which would map code which was _mostly_ similar but not exactly identical.

## Measuring Popularity

Popularity was measured by fingerprinting every class and method in the benign sample set. This was 107916 apps taken from a variety of sources including non-english speaking markets. Labels were determined by a combination of VirusTotal and manual scanning. VirusTotal criteria were that it had 0 positives and had been scanned after 2 months of it first appearing on VirusTotal. This is to give vendors time to react to new threats. Since vendors tend to put highly sensitive models on VirusTotal, this should slightly bias the data set away from anything remotely resembling adware. However, this was remediated somewhat by adding ~50,000 samples from randomly crawled markets, many of which don't appear on VirusTotal. These were determined to be benign by scanning them with multiple Android anti-malware products.

### Popularity Distributions

Below is the class hash popularity distribution:

![class hash popularity](/images/filtering-popular-code-and-effects-on-model-accuracy/class_hash_popularity.png)

As you can see from the above figure, there is a very long tail to the distributions. This means that many classes are only seen rarely and there are only a few classes which are popular. Further, popular classes tend to be much, much more popular than non-popular classes.  These popular classes probably include empty classes and classes from popular SDKs and libraries.

Below is the method hash popularity distribution which follows a similar distribution:

![method hash popularity](/images/filtering-popular-code-and-effects-on-model-accuracy/method_hash_popularity.png)

The code for generating these hashes is available here: [graph.py](https://gist.github.com/CalebFenton/fb25d9e9e9a4450e95f89776fd40836e)

The data for these graphs is here: [android-hash-counts.7z](https://mega.nz/#!Tcc11TYC!uyY8x6IVePlW0neWh3JCf08i1Zb1rLjCQWZJzwXavGU)

### Selecting Hashes

The threshold for what constituted a "popular" class was 75, which was determined visually. In other words, I kinda eyeballed it. For methods, the threshold was 100. This means, if a class was seen 75 times or a method was seen 100 times, it should be skipped when extracting features. However, this could lead to a loss of valuable information, maybe, so I added a few features for number of classes and methods skipped to the finished model.

The code for selecting hashes is available here: [select\_popular\_hashes.py](https://gist.github.com/CalebFenton/b23550864120621a8f459caca90a1b9f)

### Top 100 Class Paths

Below is a list of the top 100 class paths, sorted by the number of times they were seen. Note, the class path is the _first_ class path seen, not a list of all class paths associated with a hash. From manual inspection, the first several classes were mostly empty and it's likely that _many_ paths map to the same hash.


|#|Hash|Count|Path|
|----|-----|----|----|
|1|-6539677568569540121|87112|Lpl/realitypump/ironsky/R;|
|2|7879150481849290686|74164|Lxr;|
|3|-2675497598636306534|70696|Lcom/realitypump/G2/G2GLSurfaceView$1;|
|4|-5007384885342135023|69153|Lcom/devilishgames/nativeextensions/admob/R$layout;|
|5|7877155950984699004|68682|Landroid/support/v4/app/SuperNotCalledException;|
|6|-7642079844424071458|65704|Lcom/socialize/ui/comment/CommentDetailActivity;|
|7|4173837491973374472|62937|Landroid/support/v4/app/FragmentManagerImpl$1;|
|8|-5936151236995089740|60471|Layj;|
|9|2308415025046174030|57786|Landroid/support/v4/content/ModernAsyncTask$4;|
|10|6014432581692261376|56086|Lcom/facebook/internal/FileLruCache$2;|
|11|-4099485795817903106|55354|Lfq;|
|12|-902756984570334141|53880|Lgj;|
|13|2544757508923420717|53475|Lcom/google/android/gms/internal/i;|
|14|-3399958614432743607|51806|Landroid/support/v4/view/ViewConfigurationCompatFroyo;|
|15|6743212674057076693|51617|Landroid/support/v4/app/FragmentManagerState;|
|16|-7929451300585271506|51499|Landroid/support/v4/view/accessibility/AccessibilityManagerCompat$AccessibilityManagerIcsImpl$1;|
|17|-5808535828223012233|51454|Landroid/support/v4/view/VelocityTrackerCompatHoneycomb;|
|18|-740089631434253045|51388|Lfo;|
|19|-5945944459787456127|51047|Landroid/support/v4/app/ActivityCompatHoneycomb;|
|20|4074238650528361008|50715|Landroid/support/v4/view/ViewCompatGingerbread;|
|21|8457541340339836364|50561|Landroid/support/v4/app/ShareCompatJB;|
|22|-7602298695408306862|50458|Landroid/support/v4/util/LogWriter;|
|23|5534139342145712559|50432|Lcom/google/ads/internal/j$a$1;|
|24|-4664196824785783660|50412|Lretrofit/http/Streaming;|
|25|-989624789237851679|50381|Lpl/realitypump/ironsky/R$style;|
|26|-5573814636106502307|50345|Landroid/support/v4/view/ViewPager$Decor;|
|27|-4316020411289880285|50316|Lpl/realitypump/ironsky/BuildConfig;|
|28|2305477454044242758|50214|Landroid/support/v4/content/pm/ActivityInfoCompat;|
|29|-2217179160054368884|49972|Landroid/support/v4/app/TaskStackBuilderJellybean;|
|30|-3424197808730443223|49926|Landroid/support/v4/view/ViewGroupCompatIcs;|
|31|906934702213877563|49773|Landroid/support/v4/accessibilityservice/AccessibilityServiceInfoCompatIcs;|
|32|-8430991917603055454|49760|Landroid/support/v4/os/ParcelableCompatCreatorCallbacks;|
|33|8802143954482027195|49534|Landroid/support/v4/app/Fragment$InstantiationException;|
|34|1819905089452899121|49474|Landroid/support/v4/app/NavUtilsJB;|
|35|-2969348625307750576|49298|Landroid/support/v4/net/ConnectivityManagerCompatJellyBean;|
|36|-9115786371403805780|49239|Lcom/devilishgames/nativeextensions/admob/R$attr;|
|37|7268059251633904232|49229|Lee;|
|38|-7105014750669022041|49090|Landroid/support/v4/widget/ResourceCursorAdapter;|
|39|7889800717182803764|49026|Lcom/ormma/view/Browser$4;|
|40|-5917040822247039264|49005|Landroid/support/v4/view/VelocityTrackerCompat$BaseVelocityTrackerVersionImpl;|
|41|-5720493436074601324|48997|Landroid/support/v4/view/VelocityTrackerCompat$HoneycombVelocityTrackerVersionImpl;|
|42|-1588742293679105513|48978|Landroid/support/v4/os/ParcelableCompatCreatorHoneycombMR2Stub;|
|43|-1957521141270853035|48707|Lcom/google/ads/util/g$1;|
|44|4547472865708723651|48653|Landroid/support/v4/os/ParcelableCompat$CompatCreator;|
|45|679737453138114582|48048|Landroid/support/v4/view/KeyEventCompatHoneycomb;|
|46|7698708090613627434|47722|Landroid/support/v4/app/FragmentManager$OnBackStackChangedListener;|
|47|2814151969022890373|47699|Landroid/support/v4/content/LocalBroadcastManager$BroadcastRecord;|
|48|4341191700874667005|47672|Landroid/support/v4/content/ContextCompatJellybean;|
|49|7588347216197350949|47672|Landroid/support/v4/content/ModernAsyncTask$WorkerRunnable;|
|50|5989011503337385482|47576|Landroid/support/v4/view/ViewGroupCompat$ViewGroupCompatIcsImpl;|
|51|-8484019840654742338|47487|Landroid/support/v4/view/accessibility/AccessibilityRecordCompatJellyBean;|
|52|-7092896683557467419|47449|Lcom/ormma/controller/Defines;|
|53|-4701498684616902325|47362|Lcom/socialize/R;|
|54|530400150896447174|47339|Landroid/support/v4/view/ViewCompat$GBViewCompatImpl;|
|55|-5103961978418681524|47338|Lcom/facebook/widget/GraphObjectAdapter$3;|
|56|645745845319975076|47300|Landroid/support/v4/view/ViewPager$PagerObserver;|
|57|5597419562948276843|47252|Landroid/support/v4/app/ListFragment$1;|
|58|662320570537411526|47214|Landroid/support/v4/widget/CursorAdapter$MyDataSetObserver;|
|59|6941237681180831691|47206|Landroid/support/v4/view/ViewPager$OnPageChangeListener;|
|60|547540514056755988|47176|Landroid/support/v4/app/LoaderManager$LoaderCallbacks;|
|61|8037420148745296490|47120|Landroid/support/v4/app/Fragment$SavedState;|
|62|1431765083981923887|47057|Landroid/support/v4/view/accessibility/AccessibilityNodeProviderCompat$AccessibilityNodeProviderStubImpl;|
|63|4215379159449676179|47043|Landroid/support/v4/view/accessibility/AccessibilityRecordCompatIcs;|
|64|-1402826908821352158|47038|Landroid/support/v4/view/accessibility/AccessibilityNodeInfoCompatIcs;|
|65|6569434689745479135|47027|Landroid/support/v4/view/VelocityTrackerCompat$VelocityTrackerVersionImpl;|
|66|8380934945701363758|46987|Landroid/support/v4/widget/CursorAdapter$ChangeObserver;|
|67|-4672682710599146656|46984|Lcom/google/android/gms/drive/metadata/SearchableOrderedMetadataField;|
|68|2693836027826835029|46886|Landroid/support/v4/view/ViewPager$SimpleOnPageChangeListener;|
|69|-786851131260004710|46787|Landroid/support/v4/widget/CursorFilter$CursorFilterClient;|
|70|8667997251744534852|46787|Landroid/support/v4/app/BackStackRecord$Op;|
|71|-5475116159158576131|46748|Landroid/support/v4/view/accessibility/AccessibilityRecordCompatIcsMr1;|
|72|5002996181779391509|46723|Landroid/support/v4/view/accessibility/AccessibilityNodeInfoCompatJellyBean;|
|73|4936223830400664217|46719|Landroid/support/v4/content/LocalBroadcastManager$1;|
|74|239062217891206692|46658|Landroid/support/v4/content/Loader$OnLoadCompleteListener;|
|75|-4913400411363934046|46548|Landroid/support/v4/view/AccessibilityDelegateCompatIcs$AccessibilityDelegateBridge;|
|76|5650689994489871625|46535|Landroid/support/v4/view/AccessibilityDelegateCompatIcs$1;|
|77|4892105813053381800|46481|Landroid/support/v4/content/ModernAsyncTask$AsyncTaskResult;|
|78|6021955058463343957|46358|Landroid/support/v4/app/FragmentActivity$FragmentTag;|
|79|997731539676885091|46090|Landroid/support/v4/view/AccessibilityDelegateCompat$AccessibilityDelegateIcsImpl;|
|80|1092001379795394415|46030|Lcom/google/android/gms/common/GooglePlayServicesNotAvailableException;|
|81|6233302609198898297|46028|Landroid/support/v4/widget/SimpleCursorAdapter$ViewBinder;|
|82|855966447952261336|46024|Landroid/support/v4/widget/SimpleCursorAdapter$CursorToStringConverter;|
|83|720132315695652952|45931|Landroid/support/v4/app/FragmentState$1;|
|84|-4176127283910384678|45931|Landroid/support/v4/app/FragmentManagerState$1;|
|85|4286589477084046391|45931|Landroid/support/v4/app/BackStackState$1;|
|86|3462422210892878260|45776|Landroid/support/v4/app/ListFragment$2;|
|87|-7093935369431053865|45718|Landroid/support/v4/view/AccessibilityDelegateCompat$AccessibilityDelegateIcsImpl$1;|
|88|5066173829313712639|45678|Landroid/support/v4/app/Fragment$SavedState$1;|
|89|3904811169429859734|45667|Landroid/support/v4/content/Loader$ForceLoadContentObserver;|
|90|-2256735085640949925|45608|Landroid/support/v4/view/AccessibilityDelegateCompat$AccessibilityDelegateJellyBeanImpl;|
|91|343007254769290917|45595|Landroid/support/v4/view/ViewPager$SavedState$1;|
|92|3277445617080844160|45450|Landroid/support/v4/view/accessibility/AccessibilityNodeProviderCompat$AccessibilityNodeProviderJellyBeanImpl;|
|93|-8522484960838180971|45430|Lcom/google/ads/util/a$a;|
|94|7484484164573575266|45385|Landroid/support/v4/content/ModernAsyncTask$2;|
|95|504944274815651800|45314|Landroid/support/v4/view/ViewPager$1;|
|96|5364514390186567115|45165|Landroid/support/v4/view/accessibility/AccessibilityManagerCompatIcs$AccessibilityStateChangeListenerBridge;|
|97|-1930552062862662117|45153|Landroid/support/v4/view/AccessibilityDelegateCompatIcs;|
|98|-1303476657272658237|45088|Landroid/support/v4/view/accessibility/AccessibilityEventCompat$AccessibilityEventIcsImpl;|
|99|8194078639849743862|45063|Landroid/support/v4/app/LoaderManager;|
|100|-5328076847440050904|45023|Landroid/support/v4/view/accessibility/AccessibilityRecordCompat$AccessibilityRecordJellyBeanImpl;|

### Top 100 Method Signatures

Note, method signature is first one seen and the caveats about class paths also apply here -- specifically, many popular methods are empty or nearly empty.

|#|Hash|Count|Signature|
|----|----|-----|---------|
|1|-7642079844424071458|108157|Lcom/aaplab/android/robird/util/ThemeUtils;-><init>()V|
|2|-6539677568569540121|92107|Lcom/handmark/pulltorefresh/library/R;-><init>()V|
|3|8202678715559741078|85637|Lcom/nostra13/universalimageloader/core/LoadAndDisplayImageTask$1;-><init>(Lcom/nostra13/universalimageloader/core/LoadAndDisplayImageTask;)V|
|4|4890160260607866519|84235|Ltwitter4j/internal/logging/LoggerFactory;-><init>()V|
|5|2609197996308319503|84222|Ltwitter4j/UserStreamAdapter;-><init>()V|
|6|1118179837447239723|83255|Lcom/google/analytics/tracking/android/HitBuilder;-><init>()V|
|7|-5936151236995089740|80384|Lcom/nostra13/universalimageloader/core/assist/FlushedInputStream;-><init>(Ljava/io/InputStream;)V|
|8|-7680277393425055583|78740|Ltwitter4j/internal/http/HttpClientImpl$1;-><init>(Ltwitter4j/internal/http/HttpClientImpl;)V|
|9|-7092896683557467419|78187|Lcom/nostra13/universalimageloader/core/DefaultConfigurationFactory;-><init>()V|
|10|-4701498684616902325|78131|Ltwitter4j/auth/AuthorizationFactory;-><init>()V|
|11|-3124051984392334424|74756|Lcom/aaplab/android/robird/ui/widget/photoview/ScrollerProxy;-><init>()V|
|12|-3752891835737638512|70619|Lcom/nostra13/universalimageloader/core/ImageLoaderEngine$1;-><init>(Lcom/nostra13/universalimageloader/core/ImageLoaderEngine;Lcom/nostra13/universalimageloader/core/LoadAndDisplayImageTask;)V|
|13|7877155950984699004|69899|Landroid/support/v4/app/SuperNotCalledException;-><init>(Ljava/lang/String;)V|
|14|-5007384885342135023|69160|Lcom/aaplab/android/robird/R$xml;-><init>()V|
|15|4628587325345143326|67922|Lcom/google/analytics/tracking/android/AnalyticsGmsCoreClient$AnalyticsServiceConnection;-><init>(Lcom/google/analytics/tracking/android/AnalyticsGmsCoreClient;)V|
|16|3307061431700887237|67203|Ltwitter4j/internal/logging/SLF4JLoggerFactory;-><init>()V|
|17|-8202474463569554070|67026|Lcom/nostra13/universalimageloader/core/assist/DiscCacheUtil;-><init>()V|
|18|-9115786371403805780|66898|Lcom/urbandroid/sleep/R$attr;-><init>()V|
|19|8771223191024956139|66782|Landroid/support/v4/app/NoSaveStateFrameLayout;-><init>(Landroid/content/Context;)V|
|20|-8728496932605184659|66756|Lcom/squareup/otto/ThreadEnforcer$2;-><init>()V|
|21|336673459420538461|66275|Lcom/nostra13/universalimageloader/core/assist/deque/LinkedBlockingDeque$Itr;-><init>(Lcom/nostra13/universalimageloader/core/assist/deque/LinkedBlockingDeque;)V|
|22|5729975003762079487|66226|Lcom/nostra13/universalimageloader/core/assist/deque/LinkedBlockingDeque$Itr;-><init>(Lcom/nostra13/universalimageloader/core/assist/deque/LinkedBlockingDeque;Lcom/nostra13/universalimageloader/core/assist/deque/LinkedBlockingDeque$1;)V|
|23|-2401040049338412171|65842|Lcom/nostra13/universalimageloader/core/display/SimpleBitmapDisplayer;-><init>()V|
|24|-2929060470316057339|65614|Landroid/support/v4/database/DatabaseUtilsCompat;-><init>()V|
|25|6315692033328304976|65281|Landroid/support/v4/widget/SlidingPaneLayout$SlidingPanelLayoutImplBase;-><init>()V|
|26|-893111063096445170|65223|Lcom/nostra13/universalimageloader/core/assist/MemoryCacheUtil$1;-><init>()V|
|27|-675924626915601184|65006|Lcom/aaplab/android/robird/ui/SearchFragment$1;-><init>(Lcom/aaplab/android/robird/ui/SearchFragment;Ltwitter4j/Twitter;)V|
|28|-3847130604763184100|64027|Lcom/urbandroid/common/http/StringResponseHandlerTemplate;-><init>()V|
|29|1416323870660580531|63987|Lcom/google/analytics/tracking/android/NoopDispatcher;-><init>()V|
|30|-3784336663845911879|63855|Lcom/aaplab/android/robird/ui/widget/popupmenu/ListPopupWindowCompat$ResizePopupRunnable;-><init>(Lcom/aaplab/android/robird/ui/widget/popupmenu/ListPopupWindowCompat;)V|
|31|8297103538565484969|63801|Lcom/aaplab/android/robird/ui/widget/popupmenu/ListPopupWindowCompat$ResizePopupRunnable;-><init>(Lcom/aaplab/android/robird/ui/widget/popupmenu/ListPopupWindowCompat;Lcom/aaplab/android/robird/ui/widget/popupmenu/ListPopupWindowCompat$1;)V|
|32|4173837491973374472|63394|Lcom/handmark/pulltorefresh/library/PullToRefreshBase$1;->run()V|
|33|-945002099871771481|62319|Lcom/google/android/gms/internal/ar;-><init>()V|
|34|2544757508923420717|61319|Lcom/google/android/gms/internal/m;-><init>(Ljava/lang/String;)V|
|35|-8312584374661018230|60614|Lcom/google/android/gms/wallet/o;-><init>()V|
|36|-3011160232611646563|59302|Lcom/twitter/Extractor$1;-><init>(Lcom/twitter/Extractor;)V|
|37|3315472411031331071|59008|Lcom/handmark/pulltorefresh/library/extras/PullToRefreshWebView2$JsValueCallback;-><init>(Lcom/handmark/pulltorefresh/library/extras/PullToRefreshWebView2;)V|
|38|-5559722447751808665|58966|Lde/keyboardsurfer/android/widget/crouton/Manager$1;-><init>(Lde/keyboardsurfer/android/widget/crouton/Manager;Landroid/view/View;Lde/keyboardsurfer/android/widget/crouton/Crouton;)V|
|39|3699353597961706907|58289|Landroid/support/v4/widget/SlidingPaneLayout$SlidingPanelLayoutImplJBMR1;-><init>()V|
|40|-8046356283759202622|58235|Landroid/support/v4/widget/SlidingPaneLayout$SimplePanelSlideListener;-><init>()V|
|41|6652028519967212390|57846|Lcom/squareup/otto/Bus$2;-><init>(Lcom/squareup/otto/Bus;)V|
|42|7268059251633904232|57804|Lcom/google/android/gms/maps/model/RuntimeRemoteException;-><init>(Landroid/os/RemoteException;)V|
|43|2308415025046174030|57787|Lcom/nostra13/universalimageloader/utils/ImageSizeUtils$1;-><clinit>()V|
|44|6014432581692261376|56721|Lcom/google/analytics/tracking/android/GAServiceProxy$2;->run()V|
|45|-7564279190829922402|56698|Landroid/support/v4/content/ModernAsyncTask$InternalHandler;-><init>()V|
|46|-8522484960838180971|56075|Lcom/handmark/pulltorefresh/library/PullToRefreshScrollView;-><init>(Landroid/content/Context;)V|
|47|-8522152698274272858|56041|Landroid/support/v4/widget/SearchViewCompatIcs;-><init>()V|
|48|3363593219160481475|55914|Landroid/support/v4/app/ShareCompatICS;-><init>()V|
|49|-6926701068898159683|55805|Landroid/support/v4/content/ModernAsyncTask$InternalHandler;-><init>(Landroid/support/v4/content/ModernAsyncTask$1;)V|
|50|8721373828724890608|55392|Landroid/support/v4/app/FragmentManagerState;-><clinit>()V|
|51|4213088151749349370|55379|Landroid/support/v4/app/FragmentManagerState;-><init>()V|
|52|5389859633888440453|55377|Landroid/support/v4/app/FragmentManagerState;-><init>(Landroid/os/Parcel;)V|
|53|7889800717182803764|55343|Lcom/google/android/gms/tagmanager/o$1;->bc(Ljava/lang/String;)V|
|54|-1937623317952501810|55045|Lcom/unity3d/player/d$2;-><init>(Lcom/unity3d/player/d;Landroid/view/View;)V|
|55|-1853703880280089887|54821|Landroid/support/v4/widget/ListViewAutoScrollHelper;-><init>(Landroid/widget/ListView;)V|
|56|153996737814003651|54818|Lcom/aaplab/android/robird/util/Events$PullDownToRefreshEvent;-><init>()V|
|57|-757778117290738428|54356|Landroid/support/v4/widget/ViewDragHelper$Callback;-><init>()V|
|58|-7013990137290522882|54191|Landroid/support/v4/os/ParcelableCompatCreatorHoneycombMR2;-><init>(Landroid/support/v4/os/ParcelableCompatCreatorCallbacks;)V|
|59|-526199542608917401|54151|Lcom/urbandroid/sleep/alarmclock/DontPressWithParentLayout;-><init>(Landroid/content/Context;Landroid/util/AttributeSet;)V|
|60|8802143954482027195|54126|Landroid/support/v4/app/Fragment$InstantiationException;-><init>(Ljava/lang/String;Ljava/lang/Exception;)V|
|61|-8778086982190029168|53895|Landroid/support/v4/os/ParcelableCompatCreatorHoneycombMR2;->createFromParcel(Landroid/os/Parcel;)Ljava/lang/Object;|
|62|2590316452339699669|53895|Landroid/support/v4/os/ParcelableCompatCreatorHoneycombMR2;->createFromParcel(Landroid/os/Parcel;Ljava/lang/ClassLoader;)Ljava/lang/Object;|
|63|-902756984570334141|53880|Landroid/support/v4/os/ParcelableCompatCreatorHoneycombMR2;->newArray(I)[Ljava/lang/Object;|
|64|-8155489102404949519|53812|Lcom/google/analytics/tracking/android/FutureApis;-><init>()V|
|65|7706868487092767732|53473|Landroid/support/v4/app/ListFragment;-><init>()V|
|66|5534139342145712559|53314|Lcom/aaplab/android/robird/ui/MediaFragment$1;->onLoadingCancelled(Ljava/lang/String;Landroid/view/View;)V|
|67|-2916310843770363879|53211|Lcom/aaplab/android/robird/ui/widget/HackyViewPager;-><init>(Landroid/content/Context;Landroid/util/AttributeSet;)V|
|68|-5945944459787456127|53070|Landroid/support/v4/app/ActivityCompatHoneycomb;->invalidateOptionsMenu(Landroid/app/Activity;)V|
|69|-3589436608246049539|53070|Landroid/support/v4/app/ActivityCompatHoneycomb;->dump(Landroid/app/Activity;Ljava/lang/String;Ljava/io/FileDescriptor;Ljava/io/PrintWriter;[Ljava/lang/String;)V|
|70|-7627280748607221422|52984|Landroid/support/v4/app/FragmentPagerAdapter;-><init>(Landroid/support/v4/app/FragmentManager;)V|
|71|-8514214176211164536|52797|Lcom/google/android/gms/analytics/internal/IAnalyticsService$Stub$Proxy;-><init>(Landroid/os/IBinder;)V|
|72|8321315701923846730|52579|Lcom/google/android/gms/dynamic/a$2;-><init>(Lcom/google/android/gms/dynamic/a;Landroid/app/Activity;Landroid/os/Bundle;Landroid/os/Bundle;)V|
|73|-683162621362766186|52442|Landroid/support/v4/content/ModernAsyncTask$3;-><init>(Landroid/support/v4/content/ModernAsyncTask;Ljava/util/concurrent/Callable;)V|
|74|5818020461474712792|52404|Lcom/google/android/gms/analytics/internal/Command$1;->createFromParcel(Landroid/os/Parcel;)Lcom/google/android/gms/analytics/internal/Command;|
|75|-4927133974873939683|52197|Lcom/google/android/apps/dashclock/api/ExtensionData$1;->createFromParcel(Landroid/os/Parcel;)Lcom/google/android/apps/dashclock/api/ExtensionData;|
|76|-3399958614432743607|52112|Landroid/support/v4/view/ViewConfigurationCompatFroyo;->getScaledPagingTouchSlop(Landroid/view/ViewConfiguration;)I|
|77|-2969348625307750576|51979|Landroid/support/v4/view/KeyEventCompatEclair;->isTracking(Landroid/view/KeyEvent;)Z|
|78|-2353508751924428670|51901|Lcom/google/android/gms/games/multiplayer/realtime/RoomEntity$a;-><init>()V|
|79|-7929451300585271506|51785|Landroid/support/v4/view/accessibility/AccessibilityManagerCompat$AccessibilityManagerIcsImpl$1;->onAccessibilityStateChanged(Z)V|
|80|1657002645692441933|51685|Landroid/support/v4/view/VelocityTrackerCompatHoneycomb;->getXVelocity(Landroid/view/VelocityTracker;I)F|
|81|-5901020707757742812|51678|Landroid/support/v4/util/LogWriter;-><init>(Ljava/lang/String;)V|
|82|-780048243655997484|51617|Landroid/support/v4/app/FragmentManagerState;->describeContents()I|
|83|6743212674057076693|51617|Landroid/support/v4/app/FragmentManagerState;->writeToParcel(Landroid/os/Parcel;I)V|
|84|-736970292080916002|51616|Lcom/google/android/gms/analytics/internal/Command$1;->createFromParcel(Landroid/os/Parcel;)Ljava/lang/Object;|
|85|-5241685838796254039|51611|Lcom/google/android/gms/analytics/internal/IAnalyticsService$Stub$Proxy;->asBinder()Landroid/os/IBinder;|
|86|-5808535828223012233|51461|Landroid/support/v4/view/VelocityTrackerCompatHoneycomb;->getYVelocity(Landroid/view/VelocityTracker;I)F|
|87|-2423550744584704325|51331|Lcom/google/android/gms/internal/kc;-><clinit>()V|
|88|-6705063738206203334|51229|Lcom/google/android/gms/internal/x;-><init>(Lcom/google/android/gms/ads/AdListener;)V|
|89|-5437951835842775581|51091|Landroid/support/v4/content/ModernAsyncTask$1;-><init>()V|
|90|3400939097257307009|51055|Lcom/google/android/gms/tagmanager/cx;-><init>()V|
|91|9207932117182213076|50985|Landroid/support/v4/view/MotionEventCompatEclair;->findPointerIndex(Landroid/view/MotionEvent;I)I|
|92|8447758234808132189|50880|Landroid/support/v4/app/NavUtilsJB;->getParentActivityIntent(Landroid/app/Activity;)Landroid/content/Intent;|
|93|4074238650528361008|50793|Landroid/support/v4/view/ViewCompatGingerbread;->setOverScrollMode(Landroid/view/View;I)V|
|94|-6034617258200236949|50764|Landroid/support/v4/widget/ResourceCursorAdapter;-><init>(Landroid/content/Context;ILandroid/database/Cursor;)V|
|95|-5799817234132788985|50745|Landroid/support/v4/widget/ResourceCursorAdapter;-><init>(Landroid/content/Context;ILandroid/database/Cursor;I)V|
|96|3788487412323627274|50741|Landroid/support/v4/util/LogWriter;->flush()V|
|97|-6252821634745382551|50741|Landroid/support/v4/util/LogWriter;->flushBuilder()V|
|98|2401124206520909086|50741|Landroid/support/v4/util/LogWriter;->close()V|
|99|-4316020411289880285|50701|Lcom/handmark/pulltorefresh/library/BuildConfig;-><init>()V|
|100|8457541340339836364|50645|Landroid/support/v4/content/IntentCompatHoneycomb;->makeMainActivity(Landroid/content/ComponentName;)Landroid/content/Intent;|

## Impacts on Model Accuracy

Unfortunately, it seemed like filtering classes as described above had a negative impact on model accuracy. It's not fun to write about negative results, but it might spark some useful discussion or prevent someone else from wasting time. If you've found opposite effects, I'd love to know more details about how you implemented it.

Models were scored by selecting the top 100, 500, 1000, 5000, and 10,000 features using [mutual information scoring](http://scikit-learn.org/stable/modules/generated/sklearn.metrics.mutual_info_score.html) and then performing a 3 fold cross validation. These metrics aren't perfect since they don't represent total possible performance, but they should be good for comparison.

Below is a table of model performance scores **without** popularity filtering:

|features|recall|precision|accuracy|
|--------|------|---------|--------|
|100|0.946433740702|0.977091995842|0.967735883444|
|500|0.938567454973|0.974466194496|0.96336181481|
|1000|0.946597359445|0.978583324659|0.968432732614|
|5000|0.955420185518|0.980420266832|0.972887206921|
|10000|0.958403081067|0.981061094076|0.97440419396|


Below are the model scores **with** popularity filtering:

|features|recall|precision|accuracy|
|--------|------|---------|--------|
|100|0.860611027171|0.975812305816|0.932179212864|
|500|0.927680606418|0.97665201801|0.960124489399|
|1000|0.928871417962|0.977905691408|0.961129481943|
|5000|0.924236216036|0.979243260843|0.959762475956|
|10000|0.922213116853|0.979931426705|0.959205947827|

## Conclusion

I really expected better model performance with filtration, but a loss of ~1.5% is a fairly significant reduction in accuracy. Most of the loss in accuracy seems to be from recall since precision seems to be fairly unaffected. This suggests the model wasn't able to detect certain types of malware when popular classes were filtered. I'm really not sure why this is. I'd need to investigate specifically which samples were missed to understand more, but there are other experiments which I think would help that I'd rather do first.
 
Perhaps I could also modify thresholds to filter out only the tippy top most popular methods and classes, or reduce the threshold to filter out anything which occurs more than a dozen or so times.

## Future Work

The two things I really want to try are sample clustering and model blending. Sample clustering would allow me to cluster similar samples and reduce the total number of training samples. This would allow me to vastly scale up sample size without losing much information and without a proportional increase in training time.

Blending and stacking is training multiple models on the same data and then using the output of those models as features for a meta estimator. In this way, the strengths of different models can be combined and the individual weaknesses reduced. It's pretty common in [Kaggle](https://www.kaggle.com/competitions) competitions. I've actually done the experiments but haven't posted them here yet. Maybe I'll do that next.