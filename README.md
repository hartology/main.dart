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
              GftProgressBar (gftValue: gftValue),
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
