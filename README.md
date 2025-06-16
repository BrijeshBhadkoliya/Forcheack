router.post("/attendence/:id", async (req, res) => {
  const date = moment().format("DD-MM-YYYY");

  try {
    let currantData = [];
    try {
      currantData = await DataFind(
        `SELECT * FROM tbl_employee_attndence WHERE emplyeeId='${req.params.id}' AND date='${date}'`
      );
    } catch (err) {
      return res.status(200).json({ error: "Error fetching currantData" }, err);
    }

    let OldData = [];
    try {
      OldData = await DataFind(
        `SELECT * FROM tbl_employee_attndence WHERE emplyeeId='${req.params.id}' AND date!='${date}'`
      );
    } catch (err) {
      return res
        .status(200)
        .json({ error: "Error fetching old attendance data" });
    }

    if (OldData[0]) {
      OldData[0].all_time =
        typeof OldData[0].all_time === "string"
          ? JSON.parse(OldData[0].all_time)
          : OldData[0].all_time;
    }

    // ************** Missing ClockOut Condition *************

    // console.log("OldData[0]", OldData[OldData.length - 1]);

    let MissOut = false;
    if (OldData.length > 0) {
      if (OldData[OldData.length - 1].clockOut_time === "") {
        MissOut = true;
        if (
          OldData[OldData.length - 1].all_time[
            OldData[OldData.length - 1].all_time.length - 1
          ].status === 2
        ) {
          let BreakEntry =
            OldData[OldData.length - 1].all_time[
              OldData[OldData.length - 1].all_time.length - 1
            ];
          console.log("BreakEntry", BreakEntry);

          return res.send({
            Break: "00:00:00",
            WorkingTime: "00:00:00",
            MissingBreakOut: MissOut,
            BreakOutEntry: BreakEntry,
          });
        }
        return res.send({
          Break: "00:00:00",
          WorkingTime: "00:00:00",
          MissingClockOut: MissOut,
          OldData: OldData[OldData.length - 1],
        });
      }
    }
    // ************** Missing ClockOut Condition *************

    let lastDate = "";

    if (OldData.length > 0) {
      lastDate = OldData[OldData.length - 1].date;
    }

    let NextDate = moment(lastDate, "DD-MM-YYYY")
      .add(1, "days")
      .format("DD-MM-YYYY");

    const leaveVerify = await DataFind(
      `SELECT * FROM tbl_leave_resons WHERE emp_Id = '${req.params.id}' AND start_date = '${NextDate}'`
    );
    console.log("leaveVerify", leaveVerify);

    let processedDates = [];
    let date2 = "";
    let date1 = "";
    let diffDays = "";
    let lastEntry = "";

    if (leaveVerify.length > 0) {
      const leaveStart = moment(leaveVerify[0].start_date, "DD-MM-YYYY");
      const leaveEnd = moment(leaveVerify[0].end_date, "DD-MM-YYYY");
      const leaveDiffDays = leaveEnd.diff(leaveStart, "days");

      let leaveDates = [leaveVerify[0].start_date];
      for (let i = 0; i < leaveDiffDays; i++) {
        leaveDates.push(
          moment(leaveDates[i], "DD-MM-YYYY")
            .add(1, "days")
            .format("DD-MM-YYYY")
        );
      }

      // weekends verify in leave
      const finalLeaveDates = [];

      for (const val of leaveDates) {
        const dateObj = new Date(val.split("-").reverse().join("-"));
        const dayName = dateObj.toLocaleDateString("en-US", {
          weekday: "long",
        });

        let isWeekend = false;

        try {
          const weekendDay = await DataFind(
            `SELECT * FROM tbl_weekend WHERE days='${dayName}'`
          );

          if (weekendDay.length > 0) {
            const weekEnds = weekendDay[0].weeks.split(",").map(Number);
            const dayCount = Math.ceil(dateObj.getUTCDate() / 7);
            if (weekEnds.includes(dayCount)) {
              finalLeaveDates.push({ date: val, type: "weekend" });
              isWeekend = true;
            }
          }
        } catch (err) {
          console.log("Error fetching weekend data", err);
        }

        if (!isWeekend) {
          finalLeaveDates.push({ date: val, type: "leave" });
        }
      }
      console.log("finalLeaveDates", finalLeaveDates);

      // Insert leave or weekend
      let lastEntry = "";
      const processedDates = [];

      for (const item of finalLeaveDates) {
        const entryDate = item.date;
        const CurrantEntry = moment().format("DD-MM-YYYY");

        if (CurrantEntry === entryDate) continue;

        if (item.type === "leave") {
          const result = await DataInsert(
            `tbl_employee_attndence`,
            `emplyeeId, date, all_time, clockIn_time, clockOut_time, break_time, productive_time, extra_time, total_time, clockIn_ip, clockOut_ip, day_status, attendens_status, leave_resone_id, leave_status`,
            `'${req.params.id}', '${entryDate}', '[]', '-', '-', '-', '-', '-', '-', '-', '-', 'FD', 'A', '${leaveVerify[0].id}', '${leaveVerify[0].leave_status}'`,
            req.hostname,
            req.protocol
          );
          if (result == -1) {
            req.flash("errors", process.env.dataerror);
            return res.redirect("/valid_license");
          }
        } else if (item.type === "weekend") {
          console.log("entryDate", entryDate);

          const result = await DataInsert(
            `tbl_employee_attndence`,
            `emplyeeId, date, all_time, clockIn_time, clockOut_time, break_time, productive_time, extra_time, total_time, clockIn_ip, clockOut_ip, day_status, attendens_status, leave_resone_id`,
            `'${req.params.id}', '${entryDate}', '[]', '-', '-', '-', '-', '-', '-', '-', '-', 'WK', '-', '-'`,
            req.hostname,
            req.protocol
          );
          if (result == -1) {
            req.flash("errors", process.env.dataerror);
            return res.redirect("/valid_license");
          }
        }

        lastEntry = entryDate;
        if (entryDate !== CurrantEntry) {
          processedDates.push(entryDate);
        }
      }

      // Final date difference calculation
      date2 = moment(lastEntry, "DD-MM-YYYY");
      date1 = moment(date, "DD-MM-YYYY");
      diffDays = date1.diff(date2, "days");
    } else {
      console.log("lastEntry", lastEntry);

      const NextDate = moment(lastDate, "DD-MM-YYYY")
        .add(1, "days")
        .format("DD-MM-YYYY");

      const nextDateObj = new Date(NextDate.split("-").reverse().join("-"));
      const dayName = nextDateObj.toLocaleDateString("en-US", {
        weekday: "long",
      });

      let weekendDay = [];

      try {
        weekendDay = await DataFind(
          `SELECT * FROM tbl_weekend WHERE days='${dayName}'`
        );
      } catch (err) {
        console.log("Weekend check error:", err);
      }

      const today = new Date();
      today.setHours(0, 0, 0, 0);

      const dateToCheck = new Date(nextDateObj);
      dateToCheck.setHours(0, 0, 0, 0);

      let isWeekend = false;

      if (weekendDay.length > 0) {
        let weekEnds = weekendDay[0].weeks.split(",").map(Number);
        const dayCount = Math.ceil(nextDateObj.getUTCDate() / 7);
        console.log("dayCount", dayCount);
        if (weekEnds.includes(dayCount)) {
          isWeekend = true;
        }
      }

      if (isWeekend && dateToCheck.getTime() !== today.getTime()) {
        await DataInsert(
          `tbl_employee_attndence`,
          `emplyeeId, date, all_time, clockIn_time, clockOut_time, break_time, productive_time, extra_time, total_time, clockIn_ip, clockOut_ip, day_status, attendens_status, leave_resone_id`,
          `'${req.params.id}', '${NextDate}', '[]', '-', '-', '-', '-', '-', '-', '-', '-', 'WK', '-', '-'`,
          req.hostname,
          req.protocol
        );
        lastEntry = NextDate;
      } else {
        lastEntry = lastDate;
      }

      date2 = moment(lastEntry, "DD-MM-YYYY");
      date1 = moment(date, "DD-MM-YYYY");
      diffDays = date1.diff(date2, "days");
    }

    let validateCode =
      processedDates.length > 0 ? processedDates.length : diffDays - 1;

    if (validateCode > 0) {
      let UpdatedData = [];
      try {
        UpdatedData = await DataFind(
          `SELECT * FROM tbl_employee_attndence WHERE emplyeeId='${req.params.id}' AND date!='${date}'`
        );
      } catch (err) {
        return res
          .status(200)
          .json({ error: "Error fetching old attendance data" });
      }

      if (UpdatedData.length > 0) {
        UpdatedData[0].all_time =
          typeof UpdatedData[0].all_time === "string"
            ? JSON.parse(UpdatedData[0].all_time)
            : UpdatedData[0].all_time;
      }

      // weekEnd Inssert Data
      let lastweeekend = "";
      let currantdate = "";
      let DatesArr = [];
      let NextDateDay = "";
      if (UpdatedData.length > 0) {
        lastweeekend = moment(
          UpdatedData[UpdatedData.length - 1]?.date,
          "DD-MM-YYYY"
        );
        currantdate = moment(date, "DD-MM-YYYY");
        DatesArr = [];
        NextDateDay = lastweeekend.clone().add(1, "days");
      }

      if (NextDateDay !== "") {
        while (NextDateDay.isSameOrBefore(currantdate)) {
          DatesArr.push(NextDateDay.format("DD-MM-YYYY"));
          NextDateDay.add(1, "days");
        }
      }

      const missingDatesWeekend = await Promise.all(
        DatesArr.map(async (val) => {
          const datedayfind = val.split("-").reverse().join("-");
          const dateObj = new Date(datedayfind);
          const dayName = dateObj.toLocaleDateString("en-US", {
            weekday: "long",
          });
          // console.log(dayName, "Date", dateObj);

          let weekendDay = [];

          try {
            weekendDay = await DataFind(
              `SELECT * FROM tbl_weekend WHERE days='${dayName}'`
            );
          } catch (err) {
            return res
              .status(200)
              .json({ error: "Error fetching weekendDay" }, err);
          }

          if (weekendDay.length > 0) {
            let weekEnds = weekendDay[0].weeks.split(",").map(Number);
            const dayCount = Math.ceil(dateObj.getUTCDate() / 7);
            // console.log(dayCount);

            if (weekEnds.includes(dayCount)) {
              // console.log("Weekend day, skipping:", val);
              return "weekend";
            }
          }
          return val;
        })
      );

      console.log("missingDatesWeekend",missingDatesWeekend);
      
      let weekendsInsertedDatas = false;
      let processedDatesAfterweek = [];

      missingDatesWeekend.forEach(async (val, i) => {
        if (val === "weekend" && !weekendsInsertedDatas) {
          console.log("Weekend date:", DatesArr[i]);
          let CurrantEntry = moment().format("DD-MM-YYYY");
          if (CurrantEntry !== DatesArr[i]) {
            const result = await DataInsert(
              `tbl_employee_attndence`,
              `emplyeeId, date, all_time, clockIn_time, clockOut_time, break_time, productive_time, extra_time, total_time, clockIn_ip, clockOut_ip, day_status, attendens_status, leave_resone_id`,
              `'${req.params.id}', '${
                DatesArr[i]
              }', '[]', '-', '-', '-', '-', '-', '-', '-', '-', 'WK', '-', '${"-"}'`,
              req.hostname,
              req.protocol
            );

            if (result == -1) {
              req.flash("errors", process.env.dataerror);
              return res.redirect("/valid_license");
            }
          }
        } else if (val !== "weekend") {
          weekendsInsertedDatas = true;
        }

        let CurrantEntry = moment().format("DD-MM-YYYY");
        if (val !== CurrantEntry) {
          processedDatesAfterweek.push(val);
        }
      });

      const date2 = moment(processedDatesAfterweek[0], "DD-MM-YYYY");
      const date1 = moment(date, "DD-MM-YYYY");
      const diffDays = date1.diff(date2, "days");

      if (diffDays > 0) {
        let dateArray = [];

        while (date2.isSameOrBefore(date1)) {
          dateArray.push(date2.format("DD-MM-YYYY"));
          date2.add(1, "days");
        }

        let firstDates = [];
        for (let val of dateArray) {
          if (val !== moment().format("DD-MM-YYYY")) {
            firstDates.push(val);
          }
        }

        console.log("dateArray", firstDates);

        return res.status(200).json({
          Break: "00:00:00",
          WorkingTime: "00:00:00",
          MissingDates: dateArray,
          firstDates: firstDates,
        });
      }
    } else if (currantData.length > 0 && currantData[0].clockOut_time === "") {
      const parsedAllTime = JSON.parse(currantData[0].all_time);

      const BarakCheak = parsedAllTime[parsedAllTime.length - 1].status;

      if (BarakCheak === 1) {
        let BreakTime = moment.duration(currantData[0].break_time).asSeconds();
        let setting = await DataFind(`SELECT * FROM tbl_setting LIMIT 1`);
        let hostsetBreack = moment.duration(setting[0].break_time).asSeconds();
        console.log(BreakTime);
        console.log(hostsetBreack);

        if (hostsetBreack < BreakTime) {
          console.log("maigc");

          return res.status(200).json({
            BreackOver: true,
            BreackTime: setting[0].break_time,
          });
        }
      }

      currantData[0].all_time =
        typeof currantData[0].all_time === "string"
          ? JSON.parse(currantData[0].all_time)
          : currantData[0].all_time;
      const ip = req.headers["x-forwarded-for"] || req.socket.remoteAddress;
      const all_time = currantData[0].all_time;
      let lastStatus = all_time[all_time.length - 1].status;

      if (lastStatus !== "") {
        all_time[all_time.length - 1].outtime = new Date().toISOString();
        all_time[all_time.length - 1].OutTimeIP = ip;
        let status = lastStatus === 1 ? 2 : 1;
        all_time.push({
          intime: new Date().toISOString(),
          status,
          outtime: "",
          InTimeIP: ip,
          OutTimeIP: "",
        });
      } else {
        all_time.push({
          intime: new Date().toISOString(),
          status: 1,
          outtime: "",
          InTimeIP: ip,
          OutTimeIP: "",
        });
      }

      // Working Time & Break Time
      const WorkingTime = all_time.filter((t) => t.status === 1);
      const BreakTime = all_time.filter((t) => t.status === 2);

      const calculateDuration = (arr) => {
        return arr.reduce((acc, val) => {
          let start = moment(val.intime),
            end = moment(val.outtime);
          if (start.isValid() && end.isValid()) {
            acc.add(moment.duration(end.diff(start)));
          }
          return acc;
        }, moment.duration());
      };

      let workDur = calculateDuration(WorkingTime);
      let breakDur = calculateDuration(BreakTime);

      let ProductiveTime = `${Math.floor(
        workDur.asHours()
      )}:${workDur.minutes()}:${workDur.seconds()}`;
      let totalBreakTime = `${Math.floor(
        breakDur.asHours()
      )}:${breakDur.minutes()}:${breakDur.seconds()}`;
      let clockIn_ip =
        all_time[all_time.length - 1].status === 1
          ? ip
          : currantData[0].clockIn_ip;
      try {
        const updated = await DataUpdate(
          `
          tbl_employee_attndence `,
          `all_time='${JSON.stringify(
            all_time
          )}',productive_time='${ProductiveTime}',break_time='${totalBreakTime}', clockIn_Ip='${clockIn_ip}'`,
          `emplyeeId = '${req.params.id}' AND date='${date}'`,
          req.hostname,
          req.protocol
        );

        if (updated === -1) {
          req.flash("errors", process.env.dataerror);
          return res.redirect("/valid_license");
        }

        return res.status(200).json({
          Break: totalBreakTime,
          WorkingTime: ProductiveTime,
          MissingDates: [],
        });
      } catch (err) {
        return res
          .status(200)
          .json({ error: "Error updating working/break time" });
      }
    } else if (currantData.length > 0) {
      currantData[0].all_time =
        typeof currantData[0].all_time === "string"
          ? JSON.parse(currantData[0].all_time)
          : currantData[0].all_time;

      let attendence = currantData[0];

      let all_time = attendence.all_time;
      let Laststatus = all_time[all_time.length - 1]?.status;
      const ip = req.headers["x-forwarded-for"] || req.socket.remoteAddress;

      if (Laststatus !== "") {
        all_time[all_time.length - 1].outtime = new Date().toISOString();
        all_time[all_time.length - 1].OutTimeIP = ip;
        let status = lastStatus === 1 ? 2 : 1;
        all_time.push({
          intime: new Date().toISOString(),
          status,
          outtime: "",
          InTimeIP: ip,
          OutTimeIP: "",
        });
      } else {
        all_time.push({
          intime: new Date().toISOString(),
          status: 1,
          outtime: "",
          InTimeIP: ip,
          OutTimeIP: "",
        });
      }

      try {
        const updated = await DataUpdate(
          `tbl_employee_attndence`,
          `all_time='${JSON.stringify(
            all_time
          )}',clockOut_time='',clockOut_ip='', extra_time='', total_time='', attendens_status='P'`,
          `emplyeeId = '${req.params.id}' AND date='${date}'`,
          req.hostname,
          req.protocol
        );

        if (updated === -1) {
          req.flash("errors", process.env.dataerror);
          return res.redirect("/valid_license");
        }

        return res.status(200).json({
          Break: attendence.break_time,
          WorkingTime: attendence.productive_time,
          MissingDates: [],
        });
      } catch (err) {
        return res
          .status(200)
          .json({ error: "Error updating clock-out details" });
      }
    } else {
      // No entry today
      const clockIn_time = moment().format("hh:mm:ss A");

      const ip = req.headers["x-forwarded-for"] || req.socket.remoteAddress;

      const all_time = [
        {
          intime: new Date().toISOString(),
          outtime: "",
          status: 1,
          InTimeIP: ip,
          OutTimeIP: "",
        },
      ];

      const datelocal = new Date();
      const daycount = Math.ceil(datelocal.getUTCDate() / 7);
      const dayName = datelocal.toLocaleString("en-US", { weekday: "long" });
      const weekendDay = await DataFind(
        `SELECT * FROM tbl_weekend WHERE days='${dayName}'`
      );

      let isWeekend = "";
      if (weekendDay.length > 0) {
        let weekenddays = weekendDay[0].weeks.split(",").map(Number);
        isWeekend = weekenddays.filter((val) => val === daycount);
      } else {
        console.log(`No weekend configuration found for day: ${dayName}`);
      }
      console.log(isWeekend);

      if (isWeekend.length === 0) {
        try {
          const inserted = await DataInsert(
            `tbl_employee_attndence`,
            ` emplyeeId, date, all_time, clockIn_time, clockOut_time, break_time, productive_time, extra_time, total_time, clockIn_ip, clockOut_ip, day_status,attendens_status`,
            `'${req.params.id}', '${date}', '${JSON.stringify(
              all_time
            )}', '${clockIn_time}','${""}','${""}','${""}','${""}','${""}','${ip}','${""}','FD', 'P'`,
            req.hostname,
            req.protocol
          );

          if (inserted === -1) {
            req.flash("errors", process.env.dataerror);
            return res.redirect("/valid_license");
          }

          return res.status(200).json({
            Break: "00:00:00",
            WorkingTime: "00:00:00",
            MissingDates: [],
          });
        } catch (err) {
          return res
            .status(500)
            .json({ error_msg: "Error inserting new attendance" });
        }
      } else {
        console.log("wekk");
        return res
          .status(200)
          .json({ error_msg: "Today is weekend, you did not clock in" });
      }
    }
  } catch (err) {
    console.error("Unexpected error:", err);
    res.status(200).json({ error_msg: "Internal server error", err });
  }
});
