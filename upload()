graph TD
    StartUpdate["Start update(detection)<br/>Entry point with new detection list"] --> LogStart["Log start message<br/>Logs entry message with current state & detection info."];

    LogStart --> SendNN["send_nn_detections(detection)<br/>If nn_dist_sender exists, sends raw<br/>detection start/finish via NATS."];

    SendNN --> LogState1["Log current state (locations)<br/>Logs tail, body, head track section locations."];

    LogState1 --> ClearData["clear_old_data_()<br/>Removes old entries from detected_starts/ends<br/>& previous_start/end_dist TimeSeries<br/>based on age/max_len."];

    ClearData --> CheckRetarder{"Is in retarder?<br/>Checks self.is_in_the_retarder flag"};

    CheckRetarder -- Yes --> UpInRetarder["up_in_retarder()<br/>Updates motion_module with current speed<br/>(potentially 0 if no recent sensor data).<br/>May reinitialize speed smoother.<br/>Does *not* change coordinates here."];

    CheckRetarder -- No --> AnalyzeTrust;
    UpInRetarder --> AnalyzeTrust;

    AnalyzeTrust["analysis_of_trust_in_detections(detection) -> data<br/>Compares otcep's TrackSections vs detection sectors vs alive sectors.<br/>Checks if detections overlap with neighbor otceps.<br/>Applies perspective correction to coordinates if trusted.<br/>Returns dict: {trust_in_tails, trust_in_heads, is_coasting,<br/>is_coasting_deletable, start, finish}."];

    AnalyzeTrust --> UpdateLastDet["Update last_detections<br/>Stores the current 'detection' list<br/>for the next cycle's comparison."];

    UpdateLastDet --> CheckCoasting{"Coasting scenario?<br/>(data.is_coasting)<br/>Checks flag from analysis_of_trust."};

    CheckCoasting -- Yes --> CheckCoastingDelete{"Coasting deletable?<br/>(data.is_coasting_deletable)<br/>Checks if otcep *should* be visible but isn't."};

    CheckCoastingDelete -- Yes --> CoastingDelete["coasting(is_delete=True)<br/>Calculates movement based on speed, time delta,<br/>profile accel, or KZP data (move_by_kzp).<br/>Updates coords. Increments cnt_rem.<br/>Updates motion_module, smooths speed."];

    CoastingDelete --> EndCoastingDelete["Return 0<br/>Exits update early (no new reliable detection)."];

    EndCoastingDelete --> EndUpdate["End update"];

    CheckCoastingDelete -- No --> CoastingNoDelete["coasting(is_delete=False)<br/>Calculates movement based on speed, time delta,<br/>profile accel, or KZP data (move_by_kzp).<br/>Updates coords. Does *not* increment cnt_rem.<br/>Updates motion_module, smooths speed."];

    CoastingNoDelete --> EndCoastingNoDelete["Return 0<br/>Exits update early (no new reliable detection)."];

    EndCoastingNoDelete --> EndUpdate;

    CheckCoasting -- No --> ResetKZPTimer["timer_use_kzp_for_coasting.reset()<br/>Resets countdown timer gating KZP usage<br/>during coasting (for next cycle if it coasts)."];

    ResetKZPTimer --> CheckTailTrust{"Trust tail detection?<br/>(data.trust_in_tails_detection)<br/>Checks flag from analysis_of_trust."};

    CheckTailTrust -- Yes --> ProcessTail["Process tail<br/>Calls calculate_distance_between_detections (finds gaps).<br/>Calls add_new_detected_start(data.start)<br/>-> Appends to TimeSeries, maybe feeds motion_analyzer."];

    CheckTailTrust -- No --> CheckHeadTrust;
    ProcessTail --> CheckHeadTrust;

    CheckHeadTrust{"Trust head detection?<br/>(data.trust_in_heads_detection)<br/>Checks flag from analysis_of_trust."};

    CheckHeadTrust -- Yes --> ProcessHead["Process head<br/>Calls add_new_detected_end(data.finish)<br/>-> Appends to TimeSeries & float_head list,<br/>maybe feeds motion_analyzer."];

    CheckHeadTrust -- No --> AfterDetectProcessing;
    ProcessHead --> AfterDetectProcessing;

    AfterDetectProcessing --> ResetCntRem["Reset cnt_rem = 0<br/>Resets deletion counter as<br/>a valid detection was processed."];

    ResetCntRem --> CheckFloatHead["check_float_head()<br/>Analyzes recent head detections vs track end.<br/>Sets self.is_float_head flag<br/>(True if head visible, False if near edge/obscured)."];

    CheckFloatHead --> CheckMotionState{"Determine motion update method<br/>Logic based on is_in_the_retarder<br/>and motion_module.aprox_velocity."};

    CheckMotionState -- In Retarder --> SetMotionDataRet["set_data_into_motion_modul()<br/>Feeds latest trusted start/end detection<br/>coordinates and timestamps into motion_module."];

    CheckMotionState -- Stopped (aprox_velocity == 0) --> RouteAnalysis["route_analysis()<br/>Checks motion_analyzer history for start of movement.<br/>If movement: calculates initial speed/coords,<br/>updates motion_module, reinitializes smoother,<br/>clears analyzer data."];

    CheckMotionState -- Moving (aprox_velocity != 0) --> UpCall["up(data)<br/>Updates motion_module state (time, float_head flag).<br/>Calls get_new_coordinate_v2 -> Calculates new speed/coords.<br/>Updates otcep's coords, speed.<br/>Calls check_stop -> Sets speed to 0 if below threshold.<br/>Accumulates profile data. Saves direction."];

    SetMotionDataRet --> PostMotionUpdates;
    RouteAnalysis --> PostMotionUpdates;
    UpCall --> PostMotionUpdates;

    PostMotionUpdates --> CheckResetLoco["check_reset_flag_locomotive()<br/>If head loco exists & head obscured near track end,<br/>resets loco flag & sets is_float_head=False."];

    CheckResetLoco --> CheckVirtual["if_the_car_virual()<br/>If virtual: checks time/update count.<br/>If criteria met, makes non-virtual & sends NATS msg.<br/>If not, may mark for deletion."];

    CheckVirtual --> CalcProfile["calculate_profile(self)<br/>If conditions met (e.g., single car, stable speed),<br/>triggers profile calculation logic."];

    CalcProfile --> CheckBigDiff["check_big_diff_between_detect_and_model(data)<br/>Adds current detections to history TimeSeries.<br/>Compares median of detection history to model coords.<br/>Sets flag if difference > threshold."];

    CheckBigDiff --> CorrectLen["correcting_len_otcep()<br/>Calculates current length.<br/>If < min threshold (car/loco), adjusts end_dist upwards."];

    CorrectLen --> SmoothSpeed["smoothing_speed()<br/>If not single car & not in retarder:<br/>Applies filter to self.speed.<br/>Updates self.speed_in_kmph."];

    SmoothSpeed --> UpdateTime["Update time_up, time_last_visual_contact<br/>Sets self.time_up and<br/>self.time_last_visual_contact to current time."];

    UpdateTime --> LogEnd["Log end state<br/>Logs final state message<br/>after update calculations."];

    LogEnd --> ReturnCoords["Return [start_coord, end_coord]<br/>Returns the final calculated<br/>start and end coordinates."];

    ReturnCoords --> EndUpdate;
