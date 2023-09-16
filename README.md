import 'dart:async';
import 'dart:convert';
import 'dart:io' show Directory, File, Platform;
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
import 'package:run_tracker/ui/wellDoneScreen/WellDoneScreen.dart';
import 'package:run_tracker/utils/Color.dart';
import 'package:run_tracker/utils/Constant.dart';
import 'package:run_tracker/utils/Debug.dart';
import 'package:run_tracker/utils/Preference.dart';
import 'package:run_tracker/utils/Utils.dart';
import 'package:stop_watch_timer/stop_watch_timer.dart';

import 'PausePopupScreen.dart';
import 'package:run_tracker/globalFirebase.dart';

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

    setUpFirebaseListeners();
  }

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
                child: CommonTopBar(
                  Languages.of(context)!.txtRunTracker.toUpperCase(),
                  this,
                  isShowBack: isBack,
                  isShowSetting: true,
                ),
              ),
              Expanded(
                child: _mapView(fulheight, fullwidth, context),
              ),
            ],
          ),
        ),
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
                                    if (_locationSubscription != null &&
                                        _locationSubscription!.isPaused)
                                      _locationSubscription!.resume();
                                    else
                                      getLoc();
                                    // stopWatchTimer.onExecute.add(StopWatchExecute.start);
                                    stopWatchTimer.onStartTimer();
                                  });
                                });
                              } else {
                                _locationSubscription!.pause();
                                // stopWatchTimer.onExecute.add(StopWatchExecute.stop);
                                stopWatchTimer.onStopTimer();
                                setState(() {
                                  startTrack = false;
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
                                    if (result == "false") {
                                      // stopWatchTimer.onExecute.add(StopWatchExecute.reset);
                                      stopWatchTimer.onResetTimer();
                                      isBack = true;
                                    }
                                    if (result == "true") {
                                      setState(() {
                                        startTrack = true;
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
