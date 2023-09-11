import 'dart:async';
import 'dart:convert';
import 'dart:io' show Directory, File, Platform;
import 'dart:math';
import 'dart:ui' as ui;
import 'package:geolocator/geolocator.dart' as geoLocator;
import 'package:google_mobile_ads/google_mobile_ads.dart';
import 'package:intl/intl.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:location/location.dart' hide PermissionStatus;
import 'package:path_provider/path_provider.dart';
import 'package:run_tracker/ad_helper.dart';
import 'package:run_tracker/common/commonTopBar/CommonTopBar.dart';
import 'package:run_tracker/dbhelper/datamodel/RunningData.dart';
import 'package:run_tracker/interfaces/RunningStopListener.dart';
import 'package:run_tracker/interfaces/TopBarClickListener.dart';
import 'package:run_tracker/localization/language/languages.dart';
import 'package:run_tracker/ui/countdowntimer/CountdownTimerScreen.dart';
import 'package:run_tracker/ui/mapsettings/MapSettingScreen.dart';
import 'package:run_tracker/ui/startRun/energyBar.dart';
import 'package:run_tracker/ui/wellDoneScreen/WellDoneScreen.dart';
import 'package:run_tracker/utils/Color.dart';
import 'package:run_tracker/utils/Constant.dart';
import 'package:run_tracker/utils/Debug.dart';
import 'package:run_tracker/utils/Preference.dart';
import 'package:run_tracker/utils/Utils.dart';
import 'package:stop_watch_timer/stop_watch_timer.dart';

import 'package:firebase_auth/firebase_auth.dart';
import 'package:firebase_database/firebase_database.dart';

import 'PausePopupScreen.dart';

import 'package:location/location.dart';
// import 'package:sensors/sensors.dart';
import 'package:sensors_plus/sensors_plus.dart';

final FirebaseAuth _auth = FirebaseAuth.instance;
final FirebaseDatabase _database = FirebaseDatabase.instance;

class StartRunScreen extends StatefulWidget {
  static RunningStopListener? runningStopListener;

  @override
  _StartRunScreenState createState() => _StartRunScreenState();
}

class _StartRunScreenState extends State<StartRunScreen>
    with TickerProviderStateMixin
    implements TopBarClickListener, RunningStopListener {
  RunningData? runningData;

  GoogleMapController? _controller;
  Location _location = Location();

  // ignore: cancel_subscriptions
  StreamSubscription<LocationData>? _locationSubscription;
  LocationData? _currentPosition;
  LatLng _initialcameraposition = LatLng(0.5937, 0.9629);

  Map<PolylineId, Polyline> polylines = {};
  List<LatLng> polylineCoordinatesList = [];
  Set<Marker> markers = {};

  //initialize energy value
  double energyValue = 0.0; // total energy value on firebase

  double gftValue = 0.0; // the total gft value earned stored on firebase

  double gft = 0.0; //the amount of gft earned every minute
  double gftEarned = 0.0; //the amount of gft earned every minute it is = gft
  double gftSession = 0.0; // amount of gft earned per session

  LocationData? _locationData;
  double energyUsed = 0.0; // energy used per session

  Timer? _energyDecrementTimer;
  bool isMovingByLocation =
      false; // Declare a variable to store the motion status by location
  bool isMovingByAccelerometer =
      false; // Declare a variable to store the motion status by accelerometer

  @override
  void initState() {
    super.initState();

    StartRunScreen.runningStopListener = this;
    runningData = RunningData();
    stopWatchTimer = StopWatchTimer(
        mode: StopWatchMode.countUp,
        onChangeRawSecond: (value) {
          if (currentSpeed >= 1) {
            if (currentSpeed < 2.34) {
              totalLowIntenseTime += 1;
              Debug.printLog("Intensity ::::==> Low");
            } else if (currentSpeed < 4.56) {
              totalModerateIntenseTime += 1;
              Debug.printLog("Intensity ::::==> Moderate");
            } else {
              totalHighIntenseTime += 1;
              Debug.printLog("Intensity ::::==> High");
            }
          }
        },
        onChange: (value) {});

    _getPreferences();
    _loadInterstitialAd();
    // Listen for changes in the energy value
    listenForEnergyChanges();
    // Implement the initState method for LocationService
    initLocationService();
  }

  void listenForEnergyChanges() {
    final User? user = _auth.currentUser;
    if (user != null) {
      _database
          .reference()
          .child('users')
          .child(user.uid)
          .child('energy')
          .onValue
          .listen((event) {
        if (event.snapshot.value != null) {
          setState(() {
            energyValue = event.snapshot.value.toDouble();
          });
          // Delay for 2 seconds before sending new energy value to the database
          Future.delayed(const Duration(milliseconds: 200), () {
            // Send new energy value to database
            _database
                .reference()
                .child('users')
                .child(user.uid)
                .child('energy')
                .set(energyValue);
          });
        }
      });

      // Add another listener for gft
      _database
          .reference()
          .child('users')
          .child(user.uid)
          .child('gft')
          .onValue
          .listen((event) {
        if (event.snapshot.value != null) {
          setState(() {
            gftValue = event.snapshot.value.toDouble();
          });
          // Delay for 2 seconds before sending new gft value to the database
          Future.delayed(const Duration(milliseconds: 200), () {
            // Send new gft value to database
            _database
                .reference()
                .child('users')
                .child(user.uid)
                .child('gft')
                .set(gftValue);
          });
        }
      });
    }
  }

  // Implement the location service initialization method
  void initLocationService() async {
    final location = Location();
    try {
      await location.requestPermission();
      location.onLocationChanged.listen((LocationData newLocationData) {
        setState(() {
          _locationData = newLocationData;
        });
        // Calculate the user's speed in meters per second
        double? speedMPS = _locationData!.speed;
        if (speedMPS! >= 3.0 / 3.6 && speedMPS <= 15.0 / 3.6) {
          // User is moving within the speed limit
          isMovingByLocation = true; // Update the motion status by location
        } else {
          // User is not moving within the speed limit
          isMovingByLocation = false; // Update the motion status by location
        }
      });
    } catch (e) {
      print("Error: $e");
    }
    userAccelerometerEvents.listen((UserAccelerometerEvent event) {
      setState(() {
        // Use accelerometer data to detect motion
        final acceleration = event.x + event.y + event.z;
        if (acceleration.abs() >= 0.083) {
          // User is moving, you can adjust the threshold as needed
          isMovingByAccelerometer =
              true; // Update the motion status by accelerometer
        } else {
          // User is not moving
          isMovingByAccelerometer =
              false; // Update the motion status by accelerometer
        }
      });
    });
  }

  // Implement the energy decrement timer method
  void startEnergyDecrement() {
    if (isMovingByLocation || isMovingByAccelerometer) {
      // Check if both sources indicate motion
      // User is moving by both sources, start the timer
      _energyDecrementTimer =
          Timer.periodic(Duration(minutes: 1), (Timer timer) {
        if (energyValue >= 0.2) {
          double randomValue = 0.2 + (0.6 - 0.2) * Random().nextDouble();
          setState(() {
            gft = randomValue;
            gftEarned = gft;
            gftSession += gftEarned;
            energyUsed += -0.2;

            energyValue -= 0.2;
            gftValue += gftEarned;
          });
        } else {
          setState(() {
            energyValue = 0.0;
          });
        }
      });
    }
  }

  // Implement the energy decrement timer stop method
  void _stopEnergyDecrement() {
    _energyDecrementTimer?.cancel();
  }

  double totalDistance = 0;
  double lastDistance = 0;
  double pace = 0;
  double calorisvalue = 0;
  bool setaliteEnable = false;
  bool startTrack = false;
  String? timeValue = "";
  bool isBack = true;

  double? avaragePace;
  double? finaldistance;
  double? finalspeed;

  double? weight;

  double currentSpeed = 0.0;
  int totalLowIntenseTime = 0;
  int totalModerateIntenseTime = 0;
  int totalHighIntenseTime = 0;

  late StopWatchTimer stopWatchTimer;
  bool kmSelected = true;
  InterstitialAd? _interstitialAd;
  bool _isInterstitialAdReady = false;

  _getPreferences() {
    setState(() {
      kmSelected = Preference.shared.getBool(Preference.IS_KM_SELECTED) ?? true;
      weight = (Preference.shared.getInt(Preference.WEIGHT) ?? 50).toDouble();
    });
  }

  Future<Uint8List> getBytesFromAsset(String path, int width) async {
    ByteData data = await rootBundle.load(path);
    ui.Codec codec = await ui.instantiateImageCodec(data.buffer.asUint8List(),
        targetWidth: width);
    ui.FrameInfo fi = await codec.getNextFrame();
    return (await fi.image.toByteData(format: ui.ImageByteFormat.png))!
        .buffer
        .asUint8List();
  }

  @override
  Future<void> dispose() async {
    _interstitialAd?.dispose();
    stopWatchTimer.dispose();
    _locationSubscription!.cancel();
    super.dispose();
    // fetchEnergyValue();
    listenForEnergyChanges();
    _stopEnergyDecrement();
    // _energySubscription.cancel();
  }

  void _onMapCreated(GoogleMapController _cntlr) {
    _controller = _cntlr;
    _locationSubscription = _location.onLocationChanged.listen((l) {
      _controller?.moveCamera(
        CameraUpdate.newCameraPosition(
          CameraPosition(target: LatLng(l.latitude!, l.longitude!), zoom: 20),
        ),
      );
      _locationSubscription!.cancel();
    });
  }

  @override
  Widget build(BuildContext context) {
    var fulheight = MediaQuery.of(context).size.height;
    var fullwidth = MediaQuery.of(context).size.width;
    return WillPopScope(
      onWillPop: () => customDialog(),
      child: Scaffold(
        backgroundColor: Colur.common_bg_dark,
        body: SafeArea(
          bottom: false,
          child: Column(
            mainAxisSize: MainAxisSize.max,
            mainAxisAlignment: MainAxisAlignment.start,
            children: <Widget>[
              Container(
                padding: !isBack
                    ? EdgeInsets.only(left: 15)
                    : EdgeInsets.only(left: 0),
                child: Text("Energy: ${energyValue.toStringAsFixed(2)}",
                    style: TextStyle(color: Colors.white)),
              ),
              EnergyProgressBar(energyValue: energyValue),
              Container(
                padding: !isBack
                    ? EdgeInsets.only(left: 15)
                    : EdgeInsets.only(left: 0),
                child: Text("GFT: ${gftValue.toStringAsFixed(2)}",
                    style: TextStyle(color: Colors.white)),
                // // CommonTopBar(
                //   Languages.of(context)!.txtRunTracker.toUpperCase(),
                //   this,
                //   isShowBack: isBack,
                //   isShowSetting: true,
                // ),
              ),
              if (_locationData != null)
                Text(
                    "Speed: ${(_locationData!.speed! * 3.6).toStringAsFixed(2)} km/h",
                    style: TextStyle(color: Colors.white))
              else
                Text("Speed: 0.00 km/h", style: TextStyle(color: Colors.white)),
              SizedBox(height: 20),
              Text("Energy Used: ${energyUsed.toStringAsFixed(2)}",
                  style: TextStyle(color: Colors.white)),
              SizedBox(height: 20),
              Text("gft/min.: ${gftEarned.toStringAsFixed(2)}",
                  style: TextStyle(color: Colors.white)),
              SizedBox(height: 20),
              Text("GFT this session: ${gftSession.toStringAsFixed(2)}",
                  style: TextStyle(color: Colors.white)),
              _timerAndDistance(fullwidth),
              Expanded(
                child: _mapView(fulheight, fullwidth, context),
              ),
            ],
          ),
        ),
      ),
    );
  }

  _textContainer(String text) {
    return Container(
      child: Text(
        text,
        style: TextStyle(
          fontSize: 14,
          color: Colur.txt_grey,
          fontWeight: FontWeight.w500,
        ),
      ),
    );
  }

  _timerAndDistance(double fullwidth) {
    return Container(
      padding: EdgeInsets.only(bottom: 20),
      width: fullwidth,
      color: Colur.common_bg_dark,
      child: Column(
        mainAxisSize: MainAxisSize.max,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Column(
                children: [
                  Container(
                    child: StreamBuilder<int>(
                      stream: stopWatchTimer.rawTime,
                      initialData: stopWatchTimer.rawTime.value,
                      builder: (context, snap) {
                        final value = snap.data;
                        final displayTime = value != null
                            ? StopWatchTimer.getDisplayTime(value,
                                hours: true,
                                minute: true,
                                second: true,
                                milliSecond: false)
                            : null;
                        timeValue = displayTime;
                        return Text(
                          displayTime ?? "00:00:00",
                          style: TextStyle(
                              fontSize: 60,
                              color: Colur.txt_white,
                              fontWeight: FontWeight.w400),
                        );
                      },
                    ),
                  ),
                  _textContainer(Languages.of(context)!.txtMin),
                ],
              ),
            ],
          ),
          Container(
            margin: EdgeInsets.only(left: 20, right: 20, top: 15),
            child: Row(
              children: [
                Container(
                  width: 90,
                  child: Column(
                    children: [
                      Container(
                        child: Text(
                          (kmSelected)
                              ? totalDistance.toStringAsFixed(2)
                              : Utils.kmToMile(totalDistance)
                                  .toStringAsFixed(2),
                          style: TextStyle(
                              fontSize: 32,
                              color: Colur.txt_white,
                              fontWeight: FontWeight.w400),
                        ),
                      ),
                      _textContainer((kmSelected)
                          ? Languages.of(context)!.txtKM.toUpperCase()
                          : Languages.of(context)!.txtMile.toUpperCase()),
                    ],
                  ),
                ),
                Expanded(
                  child: Center(
                    child: Column(
                      children: [
                        Container(
                          child: Text(
                            (kmSelected)
                                ? pace.toStringAsFixed(2)
                                : Utils.minPerKmToMinPerMile(pace)
                                    .toStringAsFixed(2),
                            style: TextStyle(
                                fontSize: 32,
                                color: Colur.txt_white,
                                fontWeight: FontWeight.w400),
                          ),
                        ),
                        _textContainer((kmSelected)
                            ? Languages.of(context)!
                                    .txtPaceMinPer
                                    .toUpperCase() +
                                Languages.of(context)!.txtKM.toUpperCase() +
                                ")"
                            : Languages.of(context)!
                                    .txtPaceMinPer
                                    .toUpperCase() +
                                Languages.of(context)!.txtMile.toUpperCase() +
                                ")"),
                      ],
                    ),
                  ),
                ),
                Container(
                  width: 90,
                  child: Column(
                    children: [
                      Container(
                        child: Text(
                          double.parse(calorisvalue.toStringAsFixed(1))
                              .toString(),
                          style: TextStyle(
                              fontSize: 32,
                              color: Colur.txt_white,
                              fontWeight: FontWeight.w400),
                        ),
                      ),
                      _textContainer(Languages.of(context)!.txtKCAL),
                    ],
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  _mapView(double fullheight, double fullWidth, BuildContext context) {
    return Container(
      child: Stack(
        children: [
          GoogleMap(
            initialCameraPosition:
                CameraPosition(target: _initialcameraposition, zoom: 18),
            mapType:
                setaliteEnable == true ? MapType.satellite : MapType.normal,
            onMapCreated: _onMapCreated,
            buildingsEnabled: false,
            markers: markers,
            myLocationEnabled: true,
            scrollGesturesEnabled: true,
            myLocationButtonEnabled: false,
            zoomGesturesEnabled: true,
            polylines: Set<Polyline>.of(polylines.values),
          ),
          Container(
            margin:
                EdgeInsets.only(left: 20, right: 20, bottom: fullheight * 0.03),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              mainAxisAlignment: MainAxisAlignment.end,
              children: [
                Visibility(
                  visible: !isBack,
                  child: InkWell(
                    child: Container(
                      height: 60,
                      width: 60,
                      margin: EdgeInsets.only(bottom: 10),
                      decoration: BoxDecoration(
                          shape: BoxShape.circle, color: Colors.white),
                      child: Center(
                          child: Image.asset(
                        'assets/icons/ic_setalite.png',
                        scale: 4.0,
                        color: setaliteEnable
                            ? Colur.purple_gradient_color2
                            : Colur.txt_grey,
                      )),
                    ),
                    onTap: () {
                      setState(() {
                        setaliteEnable = !setaliteEnable;
                      });
                      Debug.printLog(
                          (setaliteEnable == true) ? "Started" : "Disabled");
                    },
                  ),
                ),
                Container(
                  child: Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Visibility(
                        visible: !isBack,
                        child: InkWell(
                          onTap: () async {
                            moveCameraToUserLocation();
                          },
                          child: Container(
                            height: 60,
                            width: 60,
                            decoration: BoxDecoration(
                                shape: BoxShape.circle, color: Colors.white),
                            child: Center(
                                child: Image.asset(
                                    'assets/icons/ic_location.png',
                                    scale: 4.0,
                                    color: Colur.purple_gradient_color2)),
                          ),
                        ),
                      ),
                      Expanded(
                        child: UnconstrainedBox(
                          child: InkWell(
                            onTap: () async {
                              if (startTrack == false) {
                                Navigator.push(
                                    context,
                                    MaterialPageRoute(
                                        builder: (context) =>
                                            CountdownTimerScreen(
                                                isGreen: false)));
                                Future.delayed(Duration(milliseconds: 3900),
                                    () {
                                  setState(() {
                                    isBack = false;
                                    startTrack = true;
                                    startEnergyDecrement();
                                    if (_locationSubscription != null &&
                                        _locationSubscription!.isPaused) {
                                      _locationSubscription!.resume();
                                      startEnergyDecrement();
                                    } else
                                      getLoc();
                                    // stopWatchTimer.onExecute.add(StopWatchExecute.start);
                                    stopWatchTimer.onStartTimer();
                                  });
                                });
                              } else {
                                _locationSubscription!.pause();
                                // stopWatchTimer.onExecute.add(StopWatchExecute.stop);
                                stopWatchTimer.onStopTimer();
                                _stopEnergyDecrement();
                                setState(() {
                                  startTrack = false;
                                  _stopEnergyDecrement();
                                });
                                if (polylineCoordinatesList.length >= 1) {
                                  runningData!.eLat = polylineCoordinatesList
                                      .last.latitude
                                      .toString();
                                  runningData!.eLong = polylineCoordinatesList
                                      .last.longitude
                                      .toString();
                                } else {
                                  return showDiscardDialog();
                                }

                                await _animateToCenterofMap();

                                await calculationsForAllValues()
                                    .then((value) async {
                                  final String result = (await Navigator.push(
                                      context,
                                      PausePopupScreen(
                                          stopWatchTimer,
                                          startTrack,
                                          runningData,
                                          _controller,
                                          markers)))!;
                                  setState(() {
                                    if (_locationSubscription != null &&
                                        _locationSubscription!.isPaused)
                                      _locationSubscription!.resume();
                                    startEnergyDecrement();
                                    if (result == "false") {
                                      // stopWatchTimer.onExecute.add(StopWatchExecute.reset);
                                      stopWatchTimer.onResetTimer();
                                      isBack = true;
                                    }
                                    if (result == "true") {
                                      setState(() {
                                        startTrack = true;
                                        startEnergyDecrement();
                                        isBack = false;
                                      });
                                    }
                                  });
                                });
                              }
                            },
                            child: Container(
                              height: 60,
                              width: 160,
                              padding: EdgeInsets.only(left: 10, right: 10),
                              decoration: BoxDecoration(
                                  boxShadow: [
                                    BoxShadow(
                                      offset: Offset(0.0, 15),
                                      spreadRadius: 1,
                                      blurRadius: 50,
                                      color: Colur.purple_gradient_shadow,
                                    ),
                                  ],
                                  gradient: LinearGradient(
                                    colors: [
                                      Colur.purple_gradient_color1,
                                      Colur.purple_gradient_color2
                                    ],
                                  ),
                                  borderRadius:
                                      BorderRadius.all(Radius.circular(30))),
                              child: Center(
                                child: Row(
                                  mainAxisAlignment: MainAxisAlignment.center,
                                  children: [
                                    Flexible(
                                      child: Container(
                                        child: Text(
                                          !startTrack
                                              ? Languages.of(context)!
                                                  .txtStart
                                                  .toUpperCase()
                                              : Languages.of(context)!
                                                  .txtPause
                                                  .toUpperCase(),
                                          overflow: TextOverflow.ellipsis,
                                          style: TextStyle(
                                              fontSize: 20,
                                              color: Colur.white,
                                              fontWeight: FontWeight.w700),
                                        ),
                                      ),
                                    ),
                                    Container(
                                      margin: EdgeInsets.only(
                                          left: fullWidth * 0.015),
                                      child: Icon(
                                        startTrack
                                            ? Icons.pause
                                            : Icons.play_arrow_rounded,
                                        color: Colur.white,
                                        size: 30,
                                      ),
                                    ),
                                  ],
                                ),
                              ),
                            ),
                          ),
                        ),
                      ),
                      Visibility(
                        visible: !isBack,
                        child: InkWell(
                          child: Container(
                            height: 60,
                            width: 60,
                            margin: EdgeInsets.only(bottom: 10),
                            decoration: BoxDecoration(
                                shape: BoxShape.circle, color: Colur.txt_black),
                            child: Center(
                              child: Image.asset(
                                'assets/icons/ic_lock.png',
                                scale: 4.0,
                                color: Colur.white,
                              ),
                            ),
                          ),
                          onTap: () async {
                            AnimationController controller =
                                AnimationController(
                                    duration: const Duration(milliseconds: 400),
                                    vsync: this);

                            showDialog(
                              barrierDismissible: false,
                              context: context,
                              builder: (_) => PopUp(
                                controller: controller,
                              ),
                            );
                          },
                        ),
                      ),
                    ],
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  void showDiscardDialog() {
    showDialog(
        barrierDismissible: true,
        context: context,
        builder: (BuildContext context) {
          return AlertDialog(
            title: Text(Languages.of(context)!.txtDiscard + " ?"),
            content: Text(Languages.of(context)!.txtAlertForNoLocation),
            actions: [
              TextButton(
                child: Text(Languages.of(context)!.txtDiscard),
                onPressed: () async {
                  Navigator.of(context).pushNamedAndRemoveUntil(
                      '/homeWizardScreen', (Route<dynamic> route) => false);
                },
              ),
            ],
          );
        });
  }

  _addPolyLine() {
    print("add black polyline");
    PolylineId id = PolylineId("poly");
    Polyline polyline = Polyline(
      polylineId: id,
      color: Colors.black,
      points: polylineCoordinatesList,
      width: 4,
    );
    polylines[id] = polyline;
  }

  @override
  void setState(fn) {
    if (mounted) {
      super.setState(fn);
    }
  }

  void _loadInterstitialAd() {
    InterstitialAd.load(
      adUnitId: AdHelper.interstitialAdUnitId,
      request: AdRequest(nonPersonalizedAds: Utils.nonPersonalizedAds()),
      adLoadCallback: InterstitialAdLoadCallback(
        onAdLoaded: (ad) {
          this._interstitialAd = ad;

          ad.fullScreenContentCallback = FullScreenContentCallback(
            onAdDismissedFullScreenContent: (ad) {
              Navigator.pushAndRemoveUntil(
                  context,
                  MaterialPageRoute(
                      builder: (context) =>
                          WellDoneScreen(runningData: runningData)),
                  ModalRoute.withName("/homeWizardScreen"));
            },
          );

          _isInterstitialAdReady = true;
        },
        onAdFailedToLoad: (err) {
          print('Failed to load an interstitial ad: ${err.message}');
          _isInterstitialAdReady = false;
        },
      ),
    );
  }

  @override
  Future<void> onFinish({bool value = true}) async {
    if (_locationSubscription != null && _locationSubscription!.isPaused)
      _locationSubscription!.cancel();
    await _addEndMarker();

    Navigator.pop(context);
    runningData!.polyLine = jsonEncode(polylineCoordinatesList);

    Future.delayed(const Duration(milliseconds: 50), () async {
      final imageBytes = await _controller!.takeSnapshot();
      await saveFile(imageBytes!, "${DateTime.now().millisecond}");
    });

    // Optionally, stop the energy decrement timer when the run is finished.
    startEnergyDecrement();
  }

  Future<String?> _findLocalPath() async {
    final TargetPlatform plateform2 = Theme.of(context).platform;
    final directory = (plateform2 == TargetPlatform.android)
        ? await getExternalStorageDirectory()
        : await getApplicationDocumentsDirectory();

    return directory?.path;
  }

  Future<bool> saveFile(Uint8List imageBytes, String filename) async {
    try {
      late String _localPath;
      _localPath =
          (await _findLocalPath())! + Platform.pathSeparator + 'Download';

      final savedDir = Directory(_localPath);
      bool hasExisted = await savedDir.exists();
      if (!hasExisted) {
        savedDir.create();
        Debug.printLog(savedDir.toString());
      }

      var newFile =
          await File(savedDir.path + Platform.pathSeparator + filename + ".png")
              .create(recursive: true);
      await newFile.writeAsBytes(imageBytes);
      runningData!.imageFile = newFile;
      runningData!.image = newFile.path;

      if (_isInterstitialAdReady) {
        _interstitialAd?.show();
      } else {
        Navigator.pushAndRemoveUntil(
            context,
            MaterialPageRoute(
                builder: (context) => WellDoneScreen(runningData: runningData)),
            ModalRoute.withName("/homeWizardScreen"));
      }

      return true;
    } catch (e) {
      Debug.printLog(e.toString());
      Utils.showToast(context, e.toString());
    }
    return false;
  }

  Future<void> _addEndMarker() async {
    double? endpinlat;
    double? endpinlon;

    if (polylineCoordinatesList.length == 1) {
      endpinlat = polylineCoordinatesList.first.latitude;
      endpinlon = polylineCoordinatesList.first.longitude;
    } else {
      endpinlat = polylineCoordinatesList.last.latitude;
      endpinlon = polylineCoordinatesList.last.longitude;
    }
    LatLng endPinPosition = LatLng(endpinlat, endpinlon);

    final Uint8List markerIcon =
        await getBytesFromAsset('assets/icons/ic_map_pin_red.png', 50);
    setState(() {
      final Marker marker = Marker(
          icon: BitmapDescriptor.fromBytes(markerIcon),
          markerId: MarkerId('2'),
          position: endPinPosition);
      markers.add(marker);
    });

    Debug.printLog("marker added");

    return;
  }

  var dist = 0.0;

  getLoc() async {
    _location.changeSettings(
      accuracy: LocationAccuracy.navigation,
    );

    geoLocator.Geolocator.getPositionStream(
      locationSettings: geoLocator.LocationSettings(
          accuracy: geoLocator.LocationAccuracy.bestForNavigation),
    ).listen((position) {
      if (polylineCoordinatesList.length >= 2) {
        var speedInMps = position.speed;
        var speedKmpm = speedInMps * 0.06;
        currentSpeed = speedKmpm * 60;
        setState(() {
          pace = 1 / speedKmpm;
        });
      }
    });

    _currentPosition = await _location.getLocation();
    _initialcameraposition =
        LatLng(_currentPosition!.latitude!, _currentPosition!.longitude!);

    _locationSubscription = _location.onLocationChanged
        .listen((LocationData currentLocation) async {
      print("${currentLocation.latitude} : ${currentLocation.longitude}");
      if (startTrack) {
        if (currentLocation.latitude != null &&
            currentLocation.longitude != null) {
          if (polylineCoordinatesList.isEmpty) {
            polylineCoordinatesList.add(
                LatLng(currentLocation.latitude!, currentLocation.longitude!));
            runningData!.sLat = currentLocation.latitude!.toString();
            runningData!.sLong = currentLocation.longitude!.toString();
            LatLng startPinPosition = LatLng(double.parse(runningData!.sLat!),
                double.parse(runningData!.sLong!));
            final Uint8List markerIcon = await getBytesFromAsset(
                'assets/icons/ic_map_pin_purple.png', 50);
            setState(() {
              final Marker marker = Marker(
                  icon: BitmapDescriptor.fromBytes(markerIcon),
                  markerId: MarkerId('1'),
                  position: startPinPosition);
              markers.add(marker);
            });
          }

          lastDistance = calculateDistance(
              polylineCoordinatesList.last.latitude,
              polylineCoordinatesList.last.longitude,
              currentLocation.latitude,
              currentLocation.longitude);

          Debug.printLog(
              "previous lat: ${polylineCoordinatesList.last.latitude},prev lng: ${polylineCoordinatesList.last.longitude},curr lat: ${currentLocation.latitude},curr lng: ${currentLocation.longitude}");

          if (dist.toStringAsFixed(4) != lastDistance.toStringAsFixed(4)) {
            Debug.printLog(
                "dist: ${dist.toStringAsFixed(3)} ====> lastDistance: ${lastDistance.toStringAsFixed(3)}");
            calorisvalue = _countCalories(weight!);
          }

          dist = lastDistance;

          double conditionDistance;
          if (polylineCoordinatesList.length <= 2 && Platform.isIOS) {
            conditionDistance = 0.03;
          } else {
            conditionDistance = 0.01;
          }

          setState(() {
            if (lastDistance >= conditionDistance) {
              totalDistance += calculateDistance(
                  polylineCoordinatesList.last.latitude,
                  polylineCoordinatesList.last.longitude,
                  currentLocation.latitude,
                  currentLocation.longitude);

              polylineCoordinatesList.add(LatLng(
                  currentLocation.latitude!, currentLocation.longitude!));
              _addPolyLine();
              _controller?.moveCamera(
                CameraUpdate.newCameraPosition(
                  CameraPosition(
                      target: LatLng(currentLocation.latitude!,
                          currentLocation.longitude!),
                      zoom: 20),
                ),
              );
            } else {
              Debug.printLog("Less Than 0.1: $lastDistance");
              return;
            }
          });
        }
      }
    });
  }

  Future<void> _animateToCenterofMap() async {
    LatLngBounds boundsFromLatLngList(List<LatLng> list) {
      assert(list.isNotEmpty);
      double? x0;
      double? x1;
      double? y0;
      double? y1;
      for (LatLng latLng in list) {
        if (x0 == null) {
          x0 = x1 = latLng.latitude;
          y0 = y1 = latLng.longitude;
        } else {
          if (latLng.latitude > x1!) x1 = latLng.latitude;
          if (latLng.latitude < x0) x0 = latLng.latitude;
          if (latLng.longitude > y1!) y1 = latLng.longitude;
          if (latLng.longitude < y0!) y0 = latLng.longitude;
        }
      }
      return LatLngBounds(
          northeast: LatLng(x1!, y1!), southwest: LatLng(x0!, y0!));
    }

    jsonEncode(polylineCoordinatesList);
    LatLngBounds latLngBounds = boundsFromLatLngList(polylineCoordinatesList);
    _controller!.animateCamera(CameraUpdate.newLatLngBounds(latLngBounds, 100));
  }

  Future<void> calculationsForAllValues() async {
    avaragePace = 0;
    finaldistance = 0;
    finalspeed = 0;
    finaldistance = double.parse(totalDistance.toStringAsFixed(2));
    finalspeed = double.parse(avaragePace!.toStringAsFixed(2));
    runningData!.date = DateFormat.yMMMd().format(DateTime.now()).toString();
    int hr = int.parse(timeValue!.split(":")[0]);
    int min = int.parse(timeValue!.split(":")[1]);
    int sec = int.parse(timeValue!.split(":")[2]);
    int totalTimeInSec = (hr * 3600) + (min * 60) + (sec);
    avaragePace = totalTimeInSec / (finaldistance! * 60);

    runningData!.duration = totalTimeInSec;
    runningData!.speed = double.parse(avaragePace!.toStringAsFixed(2));
    runningData!.distance = finaldistance;
    runningData!.cal = double.parse(calorisvalue.toStringAsFixed(2));
    runningData!.lowIntenseTime = totalLowIntenseTime;
    runningData!.moderateIntenseTime = totalModerateIntenseTime;
    runningData!.highIntenseTime = totalHighIntenseTime;
  }

  var time = 0;
  double caloriesValue = 0;
  double _countCalories(double weight) {
    int hr = int.parse(timeValue!.split(":")[0]);
    int min = int.parse(timeValue!.split(":")[1]);
    int sec = int.parse(timeValue!.split(":")[2]);
    int sec2 = (hr * 3600) + (min * 60) + (sec);
    Debug.printLog("met constant: " + getMETConstant().toString());
    if (pace != 0 || totalDistance != 0) {
      caloriesValue = caloriesValue +
          ((getMETConstant() * 3.5 * weight) / 200) * ((sec2 - time) * 0.06);
    } else {
      caloriesValue = 0.0;
    }
    time = sec2;
    return caloriesValue;
  }

  double getMETConstant() {
    var runPace = Utils.minPerKmToMinPerMile(pace);
    if (runPace >= 13) {
      return 5;
    } else if (runPace >= 12) {
      return 8.3;
    } else if (runPace >= 11.5) {
      return 9;
    } else if (runPace >= 10) {
      return 8;
    } else if (runPace >= 9) {
      return 10.5;
    } else if (runPace >= 8.5) {
      return 11;
    } else if (runPace >= 8) {
      return 11.5;
    } else if (runPace >= 7.5) {
      return 11.8;
    } else if (runPace >= 7) {
      return 12.3;
    } else if (runPace >= 6.5) {
      return 12.8;
    } else if (runPace >= 6) {
      return 14.5;
    } else if (runPace >= 5.5) {
      return 16;
    } else if (runPace >= 5) {
      return 19;
    } else if (runPace >= 4.6) {
      return 19.8;
    } else if (runPace >= 4.3) {
      return 23;
    } else {
      return 8;
    }
  }

  double calculateDistance(lat1, lon1, lat2, lon2) {
    return (geoLocator.GeolocatorPlatform.instance
            .distanceBetween(lat1, lon1, lat2, lon2)) /
        1000;
  }

  @override
  void onTopBarClick(String name, {bool value = true}) {
    if (name == Constant.STR_BACK) {
      Navigator.of(context).pushNamedAndRemoveUntil(
          '/homeWizardScreen', (Route<dynamic> route) => false);
    }
    if (name == Constant.STR_SETTING) {
      Navigator.push(context,
              MaterialPageRoute(builder: (context) => MapSettingScreen()))
          .then((value) {
        _getPreferences();
      });
    }
  }

  Future<void> moveCameraToUserLocation() async {
    try {
      LatLng newCurrentlatLong = LatLng(polylineCoordinatesList.last.latitude,
          polylineCoordinatesList.last.longitude);
      _controller!.moveCamera(CameraUpdate.newLatLng(newCurrentlatLong));
    } on Exception catch (e) {
      Utils.showToast(context, "Can't Locate To your Location:$e");
      Debug.printLog(e.toString());
    }
  }

  customDialog() async {
    _locationSubscription!.pause();
    // stopWatchTimer.onExecute.add(StopWatchExecute.stop);
    stopWatchTimer.onStopTimer();
    setState(() {
      startTrack = false;
      _stopEnergyDecrement();
    });
    if (polylineCoordinatesList.length >= 1) {
      runningData!.eLat = polylineCoordinatesList.last.latitude.toString();
      runningData!.eLong = polylineCoordinatesList.last.longitude.toString();
    } else {
      return showDiscardDialog();
    }

    await _animateToCenterofMap();

    await calculationsForAllValues().then((value) async {
      final String result = (await Navigator.push(
          context,
          PausePopupScreen(
              stopWatchTimer, startTrack, runningData, _controller, markers)))!;
      setState(() {
        if (_locationSubscription != null && _locationSubscription!.isPaused)
          _locationSubscription!.resume();
        if (result == "false") {
          // stopWatchTimer.onExecute.add(StopWatchExecute.reset);
          stopWatchTimer.onResetTimer();
          isBack = true;
        }
        if (result == "true") {
          setState(() {
            startTrack = true;
            startEnergyDecrement();
            isBack = false;
          });
        }
      });
    });
  }
}

class PopUp extends StatefulWidget {
  final AnimationController? controller;
  final bool lockMode;

  PopUp({this.controller, this.lockMode = false});

  @override
  State<StatefulWidget> createState() => PopUpState();
}

class PopUpState extends State<PopUp> {
  double size = 80;

  @override
  void initState() {
    super.initState();
    widget.controller?.duration = Duration(seconds: 3);
    widget.controller?.reverseDuration = Duration(milliseconds: 500);
    widget.controller?.addListener(() {
      setState(() {});
    });
  }

  @override
  Widget build(BuildContext context) {
    return WillPopScope(
      onWillPop: () async => false,
      child: Container(
        child: Container(
          margin: EdgeInsets.only(bottom: 0),
          child: Scaffold(
            backgroundColor: Colur.transparent,
            body: Container(
              margin: EdgeInsets.only(bottom: 120),
              alignment: Alignment.bottomCenter,
              child: Stack(
                alignment: Alignment.center,
                children: <Widget>[
                  Container(
                    width: size,
                    height: size,
                    child: CircularProgressIndicator(
                      value: 2.0,
                      strokeWidth: 7,
                      valueColor: AlwaysStoppedAnimation<Color>(
                          Colur.purple_Lock_screen),
                    ),
                  ),
                  Container(
                    width: size,
                    height: size,
                    child: CircularProgressIndicator(
                      value: widget.controller?.value,
                      strokeWidth: 7,
                      valueColor: AlwaysStoppedAnimation<Color>(Colur.white),
                    ),
                  ),
                  GestureDetector(
                    onTapDown: (_) {
                      widget.controller?.forward();
                      setState(() {
                        size = 84;
                      });
                    },
                    onTapUp: (_) {
                      checkCompleted();
                    },
                    onVerticalDragEnd: (_) {
                      checkCompleted();
                    },
                    onHorizontalDragEnd: (_) {
                      checkCompleted();
                    },
                    child: Container(
                      width: 78,
                      height: 78,
                      decoration: BoxDecoration(
                        shape: BoxShape.circle,
                        gradient: LinearGradient(
                          begin: Alignment.topLeft,
                          end: Alignment.bottomRight,
                          colors: <Color>[
                            Colur.purple_gradient_color1,
                            Colur.purple_gradient_color2,
                          ],
                        ),
                      ),
                      child: Center(
                        child: Container(
                          child: Image.asset(
                            "assets/icons/ic_lock.png",
                            scale: 3.7,
                          ),
                        ),
                      ),
                    ),
                  ),
                  Container(
                    margin: EdgeInsets.only(top: 135),
                    child: Text(
                      Languages.of(context)!.txtLongPressToUnlock.toUpperCase(),
                      textAlign: ui.TextAlign.center,
                      style: TextStyle(
                          color: Colur.white,
                          fontWeight: FontWeight.w500,
                          fontSize: 17),
                    ),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }

  void checkCompleted() {
    if (widget.controller?.status == AnimationStatus.forward) {
      widget.controller?.reverse();
      setState(() {
        size = 78;
      });
    }
    if (widget.controller?.status == AnimationStatus.completed) {
      Navigator.pop(context);
    }
  }

  @override
  void dispose() {
    widget.controller?.dispose();
    super.dispose();
  }
}
import 'dart:async';
import 'dart:convert';
import 'dart:io' show Directory, File, Platform;
import 'dart:math';
import 'dart:ui' as ui;
import 'package:geolocator/geolocator.dart' as geoLocator;
import 'package:google_mobile_ads/google_mobile_ads.dart';
import 'package:intl/intl.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:location/location.dart' hide PermissionStatus;
import 'package:path_provider/path_provider.dart';
import 'package:run_tracker/ad_helper.dart';
import 'package:run_tracker/common/commonTopBar/CommonTopBar.dart';
import 'package:run_tracker/dbhelper/datamodel/RunningData.dart';
import 'package:run_tracker/interfaces/RunningStopListener.dart';
import 'package:run_tracker/interfaces/TopBarClickListener.dart';
import 'package:run_tracker/localization/language/languages.dart';
import 'package:run_tracker/ui/countdowntimer/CountdownTimerScreen.dart';
import 'package:run_tracker/ui/mapsettings/MapSettingScreen.dart';
import 'package:run_tracker/ui/startRun/energyBar.dart';
import 'package:run_tracker/ui/wellDoneScreen/WellDoneScreen.dart';
import 'package:run_tracker/utils/Color.dart';
import 'package:run_tracker/utils/Constant.dart';
import 'package:run_tracker/utils/Debug.dart';
import 'package:run_tracker/utils/Preference.dart';
import 'package:run_tracker/utils/Utils.dart';
import 'package:stop_watch_timer/stop_watch_timer.dart';

import 'package:firebase_auth/firebase_auth.dart';
import 'package:firebase_database/firebase_database.dart';

import 'PausePopupScreen.dart';

import 'package:location/location.dart';
// import 'package:sensors/sensors.dart';
import 'package:sensors_plus/sensors_plus.dart';

final FirebaseAuth _auth = FirebaseAuth.instance;
final FirebaseDatabase _database = FirebaseDatabase.instance;

class StartRunScreen extends StatefulWidget {
  static RunningStopListener? runningStopListener;

  @override
  _StartRunScreenState createState() => _StartRunScreenState();
}

class _StartRunScreenState extends State<StartRunScreen>
    with TickerProviderStateMixin
    implements TopBarClickListener, RunningStopListener {
  RunningData? runningData;

  GoogleMapController? _controller;
  Location _location = Location();

  // ignore: cancel_subscriptions
  StreamSubscription<LocationData>? _locationSubscription;
  LocationData? _currentPosition;
  LatLng _initialcameraposition = LatLng(0.5937, 0.9629);

  Map<PolylineId, Polyline> polylines = {};
  List<LatLng> polylineCoordinatesList = [];
  Set<Marker> markers = {};

  //initialize energy value
  double energyValue = 0.0; // total energy value on firebase

  double gftValue = 0.0; // the total gft value earned stored on firebase

  double gft = 0.0; //the amount of gft earned every minute
  double gftEarned = 0.0; //the amount of gft earned every minute it is = gft
  double gftSession = 0.0; // amount of gft earned per session

  LocationData? _locationData;
  double energyUsed = 0.0; // energy used per session

  Timer? _energyDecrementTimer;
  bool isMovingByLocation =
      false; // Declare a variable to store the motion status by location
  bool isMovingByAccelerometer =
      false; // Declare a variable to store the motion status by accelerometer

  @override
  void initState() {
    super.initState();

    StartRunScreen.runningStopListener = this;
    runningData = RunningData();
    stopWatchTimer = StopWatchTimer(
        mode: StopWatchMode.countUp,
        onChangeRawSecond: (value) {
          if (currentSpeed >= 1) {
            if (currentSpeed < 2.34) {
              totalLowIntenseTime += 1;
              Debug.printLog("Intensity ::::==> Low");
            } else if (currentSpeed < 4.56) {
              totalModerateIntenseTime += 1;
              Debug.printLog("Intensity ::::==> Moderate");
            } else {
              totalHighIntenseTime += 1;
              Debug.printLog("Intensity ::::==> High");
            }
          }
        },
        onChange: (value) {});

    _getPreferences();
    _loadInterstitialAd();
    // Listen for changes in the energy value
    listenForEnergyChanges();
    // Implement the initState method for LocationService
    initLocationService();
  }

  void listenForEnergyChanges() {
    final User? user = _auth.currentUser;
    if (user != null) {
      _database
          .reference()
          .child('users')
          .child(user.uid)
          .child('energy')
          .onValue
          .listen((event) {
        if (event.snapshot.value != null) {
          setState(() {
            energyValue = event.snapshot.value.toDouble();
          });
          // Delay for 2 seconds before sending new energy value to the database
          Future.delayed(const Duration(milliseconds: 200), () {
            // Send new energy value to database
            _database
                .reference()
                .child('users')
                .child(user.uid)
                .child('energy')
                .set(energyValue);
          });
        }
      });

      // Add another listener for gft
      _database
          .reference()
          .child('users')
          .child(user.uid)
          .child('gft')
          .onValue
          .listen((event) {
        if (event.snapshot.value != null) {
          setState(() {
            gftValue = event.snapshot.value.toDouble();
          });
          // Delay for 2 seconds before sending new gft value to the database
          Future.delayed(const Duration(milliseconds: 200), () {
            // Send new gft value to database
            _database
                .reference()
                .child('users')
                .child(user.uid)
                .child('gft')
                .set(gftValue);
          });
        }
      });
    }
  }

  // Implement the location service initialization method
  void initLocationService() async {
    final location = Location();
    try {
      await location.requestPermission();
      location.onLocationChanged.listen((LocationData newLocationData) {
        setState(() {
          _locationData = newLocationData;
        });
        // Calculate the user's speed in meters per second
        double? speedMPS = _locationData!.speed;
        if (speedMPS! >= 3.0 / 3.6 && speedMPS <= 15.0 / 3.6) {
          // User is moving within the speed limit
          isMovingByLocation = true; // Update the motion status by location
        } else {
          // User is not moving within the speed limit
          isMovingByLocation = false; // Update the motion status by location
        }
      });
    } catch (e) {
      print("Error: $e");
    }
    userAccelerometerEvents.listen((UserAccelerometerEvent event) {
      setState(() {
        // Use accelerometer data to detect motion
        final acceleration = event.x + event.y + event.z;
        if (acceleration.abs() >= 0.083) {
          // User is moving, you can adjust the threshold as needed
          isMovingByAccelerometer =
              true; // Update the motion status by accelerometer
        } else {
          // User is not moving
          isMovingByAccelerometer =
              false; // Update the motion status by accelerometer
        }
      });
    });
  }

  // Implement the energy decrement timer method
  void startEnergyDecrement() {
    if (isMovingByLocation || isMovingByAccelerometer) {
      // Check if both sources indicate motion
      // User is moving by both sources, start the timer
      _energyDecrementTimer =
          Timer.periodic(Duration(minutes: 1), (Timer timer) {
        if (energyValue >= 0.2) {
          double randomValue = 0.2 + (0.6 - 0.2) * Random().nextDouble();
          setState(() {
            gft = randomValue;
            gftEarned = gft;
            gftSession += gftEarned;
            energyUsed += -0.2;

            energyValue -= 0.2;
            gftValue += gftEarned;
          });
        } else {
          setState(() {
            energyValue = 0.0;
          });
        }
      });
    }
  }

  // Implement the energy decrement timer stop method
  void _stopEnergyDecrement() {
    _energyDecrementTimer?.cancel();
  }

  double totalDistance = 0;
  double lastDistance = 0;
  double pace = 0;
  double calorisvalue = 0;
  bool setaliteEnable = false;
  bool startTrack = false;
  String? timeValue = "";
  bool isBack = true;

  double? avaragePace;
  double? finaldistance;
  double? finalspeed;

  double? weight;

  double currentSpeed = 0.0;
  int totalLowIntenseTime = 0;
  int totalModerateIntenseTime = 0;
  int totalHighIntenseTime = 0;

  late StopWatchTimer stopWatchTimer;
  bool kmSelected = true;
  InterstitialAd? _interstitialAd;
  bool _isInterstitialAdReady = false;

  _getPreferences() {
    setState(() {
      kmSelected = Preference.shared.getBool(Preference.IS_KM_SELECTED) ?? true;
      weight = (Preference.shared.getInt(Preference.WEIGHT) ?? 50).toDouble();
    });
  }

  Future<Uint8List> getBytesFromAsset(String path, int width) async {
    ByteData data = await rootBundle.load(path);
    ui.Codec codec = await ui.instantiateImageCodec(data.buffer.asUint8List(),
        targetWidth: width);
    ui.FrameInfo fi = await codec.getNextFrame();
    return (await fi.image.toByteData(format: ui.ImageByteFormat.png))!
        .buffer
        .asUint8List();
  }

  @override
  Future<void> dispose() async {
    _interstitialAd?.dispose();
    stopWatchTimer.dispose();
    _locationSubscription!.cancel();
    super.dispose();
    // fetchEnergyValue();
    listenForEnergyChanges();
    _stopEnergyDecrement();
    // _energySubscription.cancel();
  }

  void _onMapCreated(GoogleMapController _cntlr) {
    _controller = _cntlr;
    _locationSubscription = _location.onLocationChanged.listen((l) {
      _controller?.moveCamera(
        CameraUpdate.newCameraPosition(
          CameraPosition(target: LatLng(l.latitude!, l.longitude!), zoom: 20),
        ),
      );
      _locationSubscription!.cancel();
    });
  }

  @override
  Widget build(BuildContext context) {
    var fulheight = MediaQuery.of(context).size.height;
    var fullwidth = MediaQuery.of(context).size.width;
    return WillPopScope(
      onWillPop: () => customDialog(),
      child: Scaffold(
        backgroundColor: Colur.common_bg_dark,
        body: SafeArea(
          bottom: false,
          child: Column(
            mainAxisSize: MainAxisSize.max,
            mainAxisAlignment: MainAxisAlignment.start,
            children: <Widget>[
              Container(
                padding: !isBack
                    ? EdgeInsets.only(left: 15)
                    : EdgeInsets.only(left: 0),
                child: Text("Energy: ${energyValue.toStringAsFixed(2)}",
                    style: TextStyle(color: Colors.white)),
              ),
              EnergyProgressBar(energyValue: energyValue),
              Container(
                padding: !isBack
                    ? EdgeInsets.only(left: 15)
                    : EdgeInsets.only(left: 0),
                child: Text("GFT: ${gftValue.toStringAsFixed(2)}",
                    style: TextStyle(color: Colors.white)),
                // // CommonTopBar(
                //   Languages.of(context)!.txtRunTracker.toUpperCase(),
                //   this,
                //   isShowBack: isBack,
                //   isShowSetting: true,
                // ),
              ),
              if (_locationData != null)
                Text(
                    "Speed: ${(_locationData!.speed! * 3.6).toStringAsFixed(2)} km/h",
                    style: TextStyle(color: Colors.white))
              else
                Text("Speed: 0.00 km/h", style: TextStyle(color: Colors.white)),
              SizedBox(height: 20),
              Text("Energy Used: ${energyUsed.toStringAsFixed(2)}",
                  style: TextStyle(color: Colors.white)),
              SizedBox(height: 20),
              Text("gft/min.: ${gftEarned.toStringAsFixed(2)}",
                  style: TextStyle(color: Colors.white)),
              SizedBox(height: 20),
              Text("GFT this session: ${gftSession.toStringAsFixed(2)}",
                  style: TextStyle(color: Colors.white)),
              _timerAndDistance(fullwidth),
              Expanded(
                child: _mapView(fulheight, fullwidth, context),
              ),
            ],
          ),
        ),
      ),
    );
  }

  _textContainer(String text) {
    return Container(
      child: Text(
        text,
        style: TextStyle(
          fontSize: 14,
          color: Colur.txt_grey,
          fontWeight: FontWeight.w500,
        ),
      ),
    );
  }

  _timerAndDistance(double fullwidth) {
    return Container(
      padding: EdgeInsets.only(bottom: 20),
      width: fullwidth,
      color: Colur.common_bg_dark,
      child: Column(
        mainAxisSize: MainAxisSize.max,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Column(
                children: [
                  Container(
                    child: StreamBuilder<int>(
                      stream: stopWatchTimer.rawTime,
                      initialData: stopWatchTimer.rawTime.value,
                      builder: (context, snap) {
                        final value = snap.data;
                        final displayTime = value != null
                            ? StopWatchTimer.getDisplayTime(value,
                                hours: true,
                                minute: true,
                                second: true,
                                milliSecond: false)
                            : null;
                        timeValue = displayTime;
                        return Text(
                          displayTime ?? "00:00:00",
                          style: TextStyle(
                              fontSize: 60,
                              color: Colur.txt_white,
                              fontWeight: FontWeight.w400),
                        );
                      },
                    ),
                  ),
                  _textContainer(Languages.of(context)!.txtMin),
                ],
              ),
            ],
          ),
          Container(
            margin: EdgeInsets.only(left: 20, right: 20, top: 15),
            child: Row(
              children: [
                Container(
                  width: 90,
                  child: Column(
                    children: [
                      Container(
                        child: Text(
                          (kmSelected)
                              ? totalDistance.toStringAsFixed(2)
                              : Utils.kmToMile(totalDistance)
                                  .toStringAsFixed(2),
                          style: TextStyle(
                              fontSize: 32,
                              color: Colur.txt_white,
                              fontWeight: FontWeight.w400),
                        ),
                      ),
                      _textContainer((kmSelected)
                          ? Languages.of(context)!.txtKM.toUpperCase()
                          : Languages.of(context)!.txtMile.toUpperCase()),
                    ],
                  ),
                ),
                Expanded(
                  child: Center(
                    child: Column(
                      children: [
                        Container(
                          child: Text(
                            (kmSelected)
                                ? pace.toStringAsFixed(2)
                                : Utils.minPerKmToMinPerMile(pace)
                                    .toStringAsFixed(2),
                            style: TextStyle(
                                fontSize: 32,
                                color: Colur.txt_white,
                                fontWeight: FontWeight.w400),
                          ),
                        ),
                        _textContainer((kmSelected)
                            ? Languages.of(context)!
                                    .txtPaceMinPer
                                    .toUpperCase() +
                                Languages.of(context)!.txtKM.toUpperCase() +
                                ")"
                            : Languages.of(context)!
                                    .txtPaceMinPer
                                    .toUpperCase() +
                                Languages.of(context)!.txtMile.toUpperCase() +
                                ")"),
                      ],
                    ),
                  ),
                ),
                Container(
                  width: 90,
                  child: Column(
                    children: [
                      Container(
                        child: Text(
                          double.parse(calorisvalue.toStringAsFixed(1))
                              .toString(),
                          style: TextStyle(
                              fontSize: 32,
                              color: Colur.txt_white,
                              fontWeight: FontWeight.w400),
                        ),
                      ),
                      _textContainer(Languages.of(context)!.txtKCAL),
                    ],
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  _mapView(double fullheight, double fullWidth, BuildContext context) {
    return Container(
      child: Stack(
        children: [
          GoogleMap(
            initialCameraPosition:
                CameraPosition(target: _initialcameraposition, zoom: 18),
            mapType:
                setaliteEnable == true ? MapType.satellite : MapType.normal,
            onMapCreated: _onMapCreated,
            buildingsEnabled: false,
            markers: markers,
            myLocationEnabled: true,
            scrollGesturesEnabled: true,
            myLocationButtonEnabled: false,
            zoomGesturesEnabled: true,
            polylines: Set<Polyline>.of(polylines.values),
          ),
          Container(
            margin:
                EdgeInsets.only(left: 20, right: 20, bottom: fullheight * 0.03),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              mainAxisAlignment: MainAxisAlignment.end,
              children: [
                Visibility(
                  visible: !isBack,
                  child: InkWell(
                    child: Container(
                      height: 60,
                      width: 60,
                      margin: EdgeInsets.only(bottom: 10),
                      decoration: BoxDecoration(
                          shape: BoxShape.circle, color: Colors.white),
                      child: Center(
                          child: Image.asset(
                        'assets/icons/ic_setalite.png',
                        scale: 4.0,
                        color: setaliteEnable
                            ? Colur.purple_gradient_color2
                            : Colur.txt_grey,
                      )),
                    ),
                    onTap: () {
                      setState(() {
                        setaliteEnable = !setaliteEnable;
                      });
                      Debug.printLog(
                          (setaliteEnable == true) ? "Started" : "Disabled");
                    },
                  ),
                ),
                Container(
                  child: Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Visibility(
                        visible: !isBack,
                        child: InkWell(
                          onTap: () async {
                            moveCameraToUserLocation();
                          },
                          child: Container(
                            height: 60,
                            width: 60,
                            decoration: BoxDecoration(
                                shape: BoxShape.circle, color: Colors.white),
                            child: Center(
                                child: Image.asset(
                                    'assets/icons/ic_location.png',
                                    scale: 4.0,
                                    color: Colur.purple_gradient_color2)),
                          ),
                        ),
                      ),
                      Expanded(
                        child: UnconstrainedBox(
                          child: InkWell(
                            onTap: () async {
                              if (startTrack == false) {
                                Navigator.push(
                                    context,
                                    MaterialPageRoute(
                                        builder: (context) =>
                                            CountdownTimerScreen(
                                                isGreen: false)));
                                Future.delayed(Duration(milliseconds: 3900),
                                    () {
                                  setState(() {
                                    isBack = false;
                                    startTrack = true;
                                    startEnergyDecrement();
                                    if (_locationSubscription != null &&
                                        _locationSubscription!.isPaused) {
                                      _locationSubscription!.resume();
                                      startEnergyDecrement();
                                    } else
                                      getLoc();
                                    // stopWatchTimer.onExecute.add(StopWatchExecute.start);
                                    stopWatchTimer.onStartTimer();
                                  });
                                });
                              } else {
                                _locationSubscription!.pause();
                                // stopWatchTimer.onExecute.add(StopWatchExecute.stop);
                                stopWatchTimer.onStopTimer();
                                _stopEnergyDecrement();
                                setState(() {
                                  startTrack = false;
                                  _stopEnergyDecrement();
                                });
                                if (polylineCoordinatesList.length >= 1) {
                                  runningData!.eLat = polylineCoordinatesList
                                      .last.latitude
                                      .toString();
                                  runningData!.eLong = polylineCoordinatesList
                                      .last.longitude
                                      .toString();
                                } else {
                                  return showDiscardDialog();
                                }

                                await _animateToCenterofMap();

                                await calculationsForAllValues()
                                    .then((value) async {
                                  final String result = (await Navigator.push(
                                      context,
                                      PausePopupScreen(
                                          stopWatchTimer,
                                          startTrack,
                                          runningData,
                                          _controller,
                                          markers)))!;
                                  setState(() {
                                    if (_locationSubscription != null &&
                                        _locationSubscription!.isPaused)
                                      _locationSubscription!.resume();
                                    startEnergyDecrement();
                                    if (result == "false") {
                                      // stopWatchTimer.onExecute.add(StopWatchExecute.reset);
                                      stopWatchTimer.onResetTimer();
                                      isBack = true;
                                    }
                                    if (result == "true") {
                                      setState(() {
                                        startTrack = true;
                                        startEnergyDecrement();
                                        isBack = false;
                                      });
                                    }
                                  });
                                });
                              }
                            },
                            child: Container(
                              height: 60,
                              width: 160,
                              padding: EdgeInsets.only(left: 10, right: 10),
                              decoration: BoxDecoration(
                                  boxShadow: [
                                    BoxShadow(
                                      offset: Offset(0.0, 15),
                                      spreadRadius: 1,
                                      blurRadius: 50,
                                      color: Colur.purple_gradient_shadow,
                                    ),
                                  ],
                                  gradient: LinearGradient(
                                    colors: [
                                      Colur.purple_gradient_color1,
                                      Colur.purple_gradient_color2
                                    ],
                                  ),
                                  borderRadius:
                                      BorderRadius.all(Radius.circular(30))),
                              child: Center(
                                child: Row(
                                  mainAxisAlignment: MainAxisAlignment.center,
                                  children: [
                                    Flexible(
                                      child: Container(
                                        child: Text(
                                          !startTrack
                                              ? Languages.of(context)!
                                                  .txtStart
                                                  .toUpperCase()
                                              : Languages.of(context)!
                                                  .txtPause
                                                  .toUpperCase(),
                                          overflow: TextOverflow.ellipsis,
                                          style: TextStyle(
                                              fontSize: 20,
                                              color: Colur.white,
                                              fontWeight: FontWeight.w700),
                                        ),
                                      ),
                                    ),
                                    Container(
                                      margin: EdgeInsets.only(
                                          left: fullWidth * 0.015),
                                      child: Icon(
                                        startTrack
                                            ? Icons.pause
                                            : Icons.play_arrow_rounded,
                                        color: Colur.white,
                                        size: 30,
                                      ),
                                    ),
                                  ],
                                ),
                              ),
                            ),
                          ),
                        ),
                      ),
                      Visibility(
                        visible: !isBack,
                        child: InkWell(
                          child: Container(
                            height: 60,
                            width: 60,
                            margin: EdgeInsets.only(bottom: 10),
                            decoration: BoxDecoration(
                                shape: BoxShape.circle, color: Colur.txt_black),
                            child: Center(
                              child: Image.asset(
                                'assets/icons/ic_lock.png',
                                scale: 4.0,
                                color: Colur.white,
                              ),
                            ),
                          ),
                          onTap: () async {
                            AnimationController controller =
                                AnimationController(
                                    duration: const Duration(milliseconds: 400),
                                    vsync: this);

                            showDialog(
                              barrierDismissible: false,
                              context: context,
                              builder: (_) => PopUp(
                                controller: controller,
                              ),
                            );
                          },
                        ),
                      ),
                    ],
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  void showDiscardDialog() {
    showDialog(
        barrierDismissible: true,
        context: context,
        builder: (BuildContext context) {
          return AlertDialog(
            title: Text(Languages.of(context)!.txtDiscard + " ?"),
            content: Text(Languages.of(context)!.txtAlertForNoLocation),
            actions: [
              TextButton(
                child: Text(Languages.of(context)!.txtDiscard),
                onPressed: () async {
                  Navigator.of(context).pushNamedAndRemoveUntil(
                      '/homeWizardScreen', (Route<dynamic> route) => false);
                },
              ),
            ],
          );
        });
  }

  _addPolyLine() {
    print("add black polyline");
    PolylineId id = PolylineId("poly");
    Polyline polyline = Polyline(
      polylineId: id,
      color: Colors.black,
      points: polylineCoordinatesList,
      width: 4,
    );
    polylines[id] = polyline;
  }

  @override
  void setState(fn) {
    if (mounted) {
      super.setState(fn);
    }
  }

  void _loadInterstitialAd() {
    InterstitialAd.load(
      adUnitId: AdHelper.interstitialAdUnitId,
      request: AdRequest(nonPersonalizedAds: Utils.nonPersonalizedAds()),
      adLoadCallback: InterstitialAdLoadCallback(
        onAdLoaded: (ad) {
          this._interstitialAd = ad;

          ad.fullScreenContentCallback = FullScreenContentCallback(
            onAdDismissedFullScreenContent: (ad) {
              Navigator.pushAndRemoveUntil(
                  context,
                  MaterialPageRoute(
                      builder: (context) =>
                          WellDoneScreen(runningData: runningData)),
                  ModalRoute.withName("/homeWizardScreen"));
            },
          );

          _isInterstitialAdReady = true;
        },
        onAdFailedToLoad: (err) {
          print('Failed to load an interstitial ad: ${err.message}');
          _isInterstitialAdReady = false;
        },
      ),
    );
  }

  @override
  Future<void> onFinish({bool value = true}) async {
    if (_locationSubscription != null && _locationSubscription!.isPaused)
      _locationSubscription!.cancel();
    await _addEndMarker();

    Navigator.pop(context);
    runningData!.polyLine = jsonEncode(polylineCoordinatesList);

    Future.delayed(const Duration(milliseconds: 50), () async {
      final imageBytes = await _controller!.takeSnapshot();
      await saveFile(imageBytes!, "${DateTime.now().millisecond}");
    });

    // Optionally, stop the energy decrement timer when the run is finished.
    startEnergyDecrement();
  }

  Future<String?> _findLocalPath() async {
    final TargetPlatform plateform2 = Theme.of(context).platform;
    final directory = (plateform2 == TargetPlatform.android)
        ? await getExternalStorageDirectory()
        : await getApplicationDocumentsDirectory();

    return directory?.path;
  }

  Future<bool> saveFile(Uint8List imageBytes, String filename) async {
    try {
      late String _localPath;
      _localPath =
          (await _findLocalPath())! + Platform.pathSeparator + 'Download';

      final savedDir = Directory(_localPath);
      bool hasExisted = await savedDir.exists();
      if (!hasExisted) {
        savedDir.create();
        Debug.printLog(savedDir.toString());
      }

      var newFile =
          await File(savedDir.path + Platform.pathSeparator + filename + ".png")
              .create(recursive: true);
      await newFile.writeAsBytes(imageBytes);
      runningData!.imageFile = newFile;
      runningData!.image = newFile.path;

      if (_isInterstitialAdReady) {
        _interstitialAd?.show();
      } else {
        Navigator.pushAndRemoveUntil(
            context,
            MaterialPageRoute(
                builder: (context) => WellDoneScreen(runningData: runningData)),
            ModalRoute.withName("/homeWizardScreen"));
      }

      return true;
    } catch (e) {
      Debug.printLog(e.toString());
      Utils.showToast(context, e.toString());
    }
    return false;
  }

  Future<void> _addEndMarker() async {
    double? endpinlat;
    double? endpinlon;

    if (polylineCoordinatesList.length == 1) {
      endpinlat = polylineCoordinatesList.first.latitude;
      endpinlon = polylineCoordinatesList.first.longitude;
    } else {
      endpinlat = polylineCoordinatesList.last.latitude;
      endpinlon = polylineCoordinatesList.last.longitude;
    }
    LatLng endPinPosition = LatLng(endpinlat, endpinlon);

    final Uint8List markerIcon =
        await getBytesFromAsset('assets/icons/ic_map_pin_red.png', 50);
    setState(() {
      final Marker marker = Marker(
          icon: BitmapDescriptor.fromBytes(markerIcon),
          markerId: MarkerId('2'),
          position: endPinPosition);
      markers.add(marker);
    });

    Debug.printLog("marker added");

    return;
  }

  var dist = 0.0;

  getLoc() async {
    _location.changeSettings(
      accuracy: LocationAccuracy.navigation,
    );

    geoLocator.Geolocator.getPositionStream(
      locationSettings: geoLocator.LocationSettings(
          accuracy: geoLocator.LocationAccuracy.bestForNavigation),
    ).listen((position) {
      if (polylineCoordinatesList.length >= 2) {
        var speedInMps = position.speed;
        var speedKmpm = speedInMps * 0.06;
        currentSpeed = speedKmpm * 60;
        setState(() {
          pace = 1 / speedKmpm;
        });
      }
    });

    _currentPosition = await _location.getLocation();
    _initialcameraposition =
        LatLng(_currentPosition!.latitude!, _currentPosition!.longitude!);

    _locationSubscription = _location.onLocationChanged
        .listen((LocationData currentLocation) async {
      print("${currentLocation.latitude} : ${currentLocation.longitude}");
      if (startTrack) {
        if (currentLocation.latitude != null &&
            currentLocation.longitude != null) {
          if (polylineCoordinatesList.isEmpty) {
            polylineCoordinatesList.add(
                LatLng(currentLocation.latitude!, currentLocation.longitude!));
            runningData!.sLat = currentLocation.latitude!.toString();
            runningData!.sLong = currentLocation.longitude!.toString();
            LatLng startPinPosition = LatLng(double.parse(runningData!.sLat!),
                double.parse(runningData!.sLong!));
            final Uint8List markerIcon = await getBytesFromAsset(
                'assets/icons/ic_map_pin_purple.png', 50);
            setState(() {
              final Marker marker = Marker(
                  icon: BitmapDescriptor.fromBytes(markerIcon),
                  markerId: MarkerId('1'),
                  position: startPinPosition);
              markers.add(marker);
            });
          }

          lastDistance = calculateDistance(
              polylineCoordinatesList.last.latitude,
              polylineCoordinatesList.last.longitude,
              currentLocation.latitude,
              currentLocation.longitude);

          Debug.printLog(
              "previous lat: ${polylineCoordinatesList.last.latitude},prev lng: ${polylineCoordinatesList.last.longitude},curr lat: ${currentLocation.latitude},curr lng: ${currentLocation.longitude}");

          if (dist.toStringAsFixed(4) != lastDistance.toStringAsFixed(4)) {
            Debug.printLog(
                "dist: ${dist.toStringAsFixed(3)} ====> lastDistance: ${lastDistance.toStringAsFixed(3)}");
            calorisvalue = _countCalories(weight!);
          }

          dist = lastDistance;

          double conditionDistance;
          if (polylineCoordinatesList.length <= 2 && Platform.isIOS) {
            conditionDistance = 0.03;
          } else {
            conditionDistance = 0.01;
          }

          setState(() {
            if (lastDistance >= conditionDistance) {
              totalDistance += calculateDistance(
                  polylineCoordinatesList.last.latitude,
                  polylineCoordinatesList.last.longitude,
                  currentLocation.latitude,
                  currentLocation.longitude);

              polylineCoordinatesList.add(LatLng(
                  currentLocation.latitude!, currentLocation.longitude!));
              _addPolyLine();
              _controller?.moveCamera(
                CameraUpdate.newCameraPosition(
                  CameraPosition(
                      target: LatLng(currentLocation.latitude!,
                          currentLocation.longitude!),
                      zoom: 20),
                ),
              );
            } else {
              Debug.printLog("Less Than 0.1: $lastDistance");
              return;
            }
          });
        }
      }
    });
  }

  Future<void> _animateToCenterofMap() async {
    LatLngBounds boundsFromLatLngList(List<LatLng> list) {
      assert(list.isNotEmpty);
      double? x0;
      double? x1;
      double? y0;
      double? y1;
      for (LatLng latLng in list) {
        if (x0 == null) {
          x0 = x1 = latLng.latitude;
          y0 = y1 = latLng.longitude;
        } else {
          if (latLng.latitude > x1!) x1 = latLng.latitude;
          if (latLng.latitude < x0) x0 = latLng.latitude;
          if (latLng.longitude > y1!) y1 = latLng.longitude;
          if (latLng.longitude < y0!) y0 = latLng.longitude;
        }
      }
      return LatLngBounds(
          northeast: LatLng(x1!, y1!), southwest: LatLng(x0!, y0!));
    }

    jsonEncode(polylineCoordinatesList);
    LatLngBounds latLngBounds = boundsFromLatLngList(polylineCoordinatesList);
    _controller!.animateCamera(CameraUpdate.newLatLngBounds(latLngBounds, 100));
  }

  Future<void> calculationsForAllValues() async {
    avaragePace = 0;
    finaldistance = 0;
    finalspeed = 0;
    finaldistance = double.parse(totalDistance.toStringAsFixed(2));
    finalspeed = double.parse(avaragePace!.toStringAsFixed(2));
    runningData!.date = DateFormat.yMMMd().format(DateTime.now()).toString();
    int hr = int.parse(timeValue!.split(":")[0]);
    int min = int.parse(timeValue!.split(":")[1]);
    int sec = int.parse(timeValue!.split(":")[2]);
    int totalTimeInSec = (hr * 3600) + (min * 60) + (sec);
    avaragePace = totalTimeInSec / (finaldistance! * 60);

    runningData!.duration = totalTimeInSec;
    runningData!.speed = double.parse(avaragePace!.toStringAsFixed(2));
    runningData!.distance = finaldistance;
    runningData!.cal = double.parse(calorisvalue.toStringAsFixed(2));
    runningData!.lowIntenseTime = totalLowIntenseTime;
    runningData!.moderateIntenseTime = totalModerateIntenseTime;
    runningData!.highIntenseTime = totalHighIntenseTime;
  }

  var time = 0;
  double caloriesValue = 0;
  double _countCalories(double weight) {
    int hr = int.parse(timeValue!.split(":")[0]);
    int min = int.parse(timeValue!.split(":")[1]);
    int sec = int.parse(timeValue!.split(":")[2]);
    int sec2 = (hr * 3600) + (min * 60) + (sec);
    Debug.printLog("met constant: " + getMETConstant().toString());
    if (pace != 0 || totalDistance != 0) {
      caloriesValue = caloriesValue +
          ((getMETConstant() * 3.5 * weight) / 200) * ((sec2 - time) * 0.06);
    } else {
      caloriesValue = 0.0;
    }
    time = sec2;
    return caloriesValue;
  }

  double getMETConstant() {
    var runPace = Utils.minPerKmToMinPerMile(pace);
    if (runPace >= 13) {
      return 5;
    } else if (runPace >= 12) {
      return 8.3;
    } else if (runPace >= 11.5) {
      return 9;
    } else if (runPace >= 10) {
      return 8;
    } else if (runPace >= 9) {
      return 10.5;
    } else if (runPace >= 8.5) {
      return 11;
    } else if (runPace >= 8) {
      return 11.5;
    } else if (runPace >= 7.5) {
      return 11.8;
    } else if (runPace >= 7) {
      return 12.3;
    } else if (runPace >= 6.5) {
      return 12.8;
    } else if (runPace >= 6) {
      return 14.5;
    } else if (runPace >= 5.5) {
      return 16;
    } else if (runPace >= 5) {
      return 19;
    } else if (runPace >= 4.6) {
      return 19.8;
    } else if (runPace >= 4.3) {
      return 23;
    } else {
      return 8;
    }
  }

  double calculateDistance(lat1, lon1, lat2, lon2) {
    return (geoLocator.GeolocatorPlatform.instance
            .distanceBetween(lat1, lon1, lat2, lon2)) /
        1000;
  }

  @override
  void onTopBarClick(String name, {bool value = true}) {
    if (name == Constant.STR_BACK) {
      Navigator.of(context).pushNamedAndRemoveUntil(
          '/homeWizardScreen', (Route<dynamic> route) => false);
    }
    if (name == Constant.STR_SETTING) {
      Navigator.push(context,
              MaterialPageRoute(builder: (context) => MapSettingScreen()))
          .then((value) {
        _getPreferences();
      });
    }
  }

  Future<void> moveCameraToUserLocation() async {
    try {
      LatLng newCurrentlatLong = LatLng(polylineCoordinatesList.last.latitude,
          polylineCoordinatesList.last.longitude);
      _controller!.moveCamera(CameraUpdate.newLatLng(newCurrentlatLong));
    } on Exception catch (e) {
      Utils.showToast(context, "Can't Locate To your Location:$e");
      Debug.printLog(e.toString());
    }
  }

  customDialog() async {
    _locationSubscription!.pause();
    // stopWatchTimer.onExecute.add(StopWatchExecute.stop);
    stopWatchTimer.onStopTimer();
    setState(() {
      startTrack = false;
      _stopEnergyDecrement();
    });
    if (polylineCoordinatesList.length >= 1) {
      runningData!.eLat = polylineCoordinatesList.last.latitude.toString();
      runningData!.eLong = polylineCoordinatesList.last.longitude.toString();
    } else {
      return showDiscardDialog();
    }

    await _animateToCenterofMap();

    await calculationsForAllValues().then((value) async {
      final String result = (await Navigator.push(
          context,
          PausePopupScreen(
              stopWatchTimer, startTrack, runningData, _controller, markers)))!;
      setState(() {
        if (_locationSubscription != null && _locationSubscription!.isPaused)
          _locationSubscription!.resume();
        if (result == "false") {
          // stopWatchTimer.onExecute.add(StopWatchExecute.reset);
          stopWatchTimer.onResetTimer();
          isBack = true;
        }
        if (result == "true") {
          setState(() {
            startTrack = true;
            startEnergyDecrement();
            isBack = false;
          });
        }
      });
    });
  }
}

class PopUp extends StatefulWidget {
  final AnimationController? controller;
  final bool lockMode;

  PopUp({this.controller, this.lockMode = false});

  @override
  State<StatefulWidget> createState() => PopUpState();
}

class PopUpState extends State<PopUp> {
  double size = 80;

  @override
  void initState() {
    super.initState();
    widget.controller?.duration = Duration(seconds: 3);
    widget.controller?.reverseDuration = Duration(milliseconds: 500);
    widget.controller?.addListener(() {
      setState(() {});
    });
  }

  @override
  Widget build(BuildContext context) {
    return WillPopScope(
      onWillPop: () async => false,
      child: Container(
        child: Container(
          margin: EdgeInsets.only(bottom: 0),
          child: Scaffold(
            backgroundColor: Colur.transparent,
            body: Container(
              margin: EdgeInsets.only(bottom: 120),
              alignment: Alignment.bottomCenter,
              child: Stack(
                alignment: Alignment.center,
                children: <Widget>[
                  Container(
                    width: size,
                    height: size,
                    child: CircularProgressIndicator(
                      value: 2.0,
                      strokeWidth: 7,
                      valueColor: AlwaysStoppedAnimation<Color>(
                          Colur.purple_Lock_screen),
                    ),
                  ),
                  Container(
                    width: size,
                    height: size,
                    child: CircularProgressIndicator(
                      value: widget.controller?.value,
                      strokeWidth: 7,
                      valueColor: AlwaysStoppedAnimation<Color>(Colur.white),
                    ),
                  ),
                  GestureDetector(
                    onTapDown: (_) {
                      widget.controller?.forward();
                      setState(() {
                        size = 84;
                      });
                    },
                    onTapUp: (_) {
                      checkCompleted();
                    },
                    onVerticalDragEnd: (_) {
                      checkCompleted();
                    },
                    onHorizontalDragEnd: (_) {
                      checkCompleted();
                    },
                    child: Container(
                      width: 78,
                      height: 78,
                      decoration: BoxDecoration(
                        shape: BoxShape.circle,
                        gradient: LinearGradient(
                          begin: Alignment.topLeft,
                          end: Alignment.bottomRight,
                          colors: <Color>[
                            Colur.purple_gradient_color1,
                            Colur.purple_gradient_color2,
                          ],
                        ),
                      ),
                      child: Center(
                        child: Container(
                          child: Image.asset(
                            "assets/icons/ic_lock.png",
                            scale: 3.7,
                          ),
                        ),
                      ),
                    ),
                  ),
                  Container(
                    margin: EdgeInsets.only(top: 135),
                    child: Text(
                      Languages.of(context)!.txtLongPressToUnlock.toUpperCase(),
                      textAlign: ui.TextAlign.center,
                      style: TextStyle(
                          color: Colur.white,
                          fontWeight: FontWeight.w500,
                          fontSize: 17),
                    ),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }

  void checkCompleted() {
    if (widget.controller?.status == AnimationStatus.forward) {
      widget.controller?.reverse();
      setState(() {
        size = 78;
      });
    }
    if (widget.controller?.status == AnimationStatus.completed) {
      Navigator.pop(context);
    }
  }

  @override
  void dispose() {
    widget.controller?.dispose();
    super.dispose();
  }
}
