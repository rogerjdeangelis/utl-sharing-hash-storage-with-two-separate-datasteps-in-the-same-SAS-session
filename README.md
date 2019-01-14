# utl-sharing-hash-storage-with-two-separate-datasteps-in-the-same-SAS-session
Sharing hash storage with two separate datasteps in the same SAS session
    Sharing hash storage with two separate datasteps in the same SAS session

    RULE: Using one HASH. Add name and dates to lab data and subset
    by Art in first datastep and Bart in second datastep

    see full solution with attributions below

     Authors

      Arthur L. Carpenter:  art@caloxy.com www.caloxy.com
      http://www.mini.pw.edu.pl/~bjablons/SASpublic/Function-Hash-Macro-sandwich.sas

      Bartosz Jablonski <yabwon@GMAIL.COM>
      Thomas Billings tebillings@gmail.com

    Suggest you use the full solution
    https://tinyurl.com/y86dr3bt
    http://www.mini.pw.edu.pl/~bjablons/SASpublic/Function-Hash-Macro-sandwich-approach-3-bidirectional-communication.sas
    or my somewhat simplified hacked not fully tested solution below ( has  log seems to work )

    SAS-L Post
    https://listserv.uga.edu/cgi-bin/wa?A2=SAS-L;948e630e.1901b

    INPUT
    =====

    Two FCMP funtions and a macro (on end)


    WORK.LAB total obs=13

     SUBJID    LAB    LABDT    LABVAL

        1      ABC    21217       1.1
        1      GHI    21246      32.0
        1      PQR    21278    5003.0
        2      ABC    20880       2.1
        2      GHI    20912      42.0
        2      PQR    20943    6003.0
        0      XYZ    20513        .
        3      ABC    21276       3.1
        3      GHI    21307      52.0
        3      PQR    21339    7003.0
        2      ABC    21306       4.1
        2      GHI    21338      62.0
        2      PQR    21369    8003.0


    WORK.ADSL total obs=3

     SUBJID    NAME      TRTSTDT    TRTENDT

        1      Art        21185      21519
        2      Bart       21217      21519
        3      Thomas     21246      21519



    EXAMPLE OUTOUT (Two datasteps sharing a HASH)
    ---------------------------------------------

    FIRST DATASTEP
    WORK.ADLB12 total obs=3

       NAME  SUBJID    LAB    LABDT    LABVAL    NAME    TRTSTDT    TRTENDT

       Art1     1      ABC    21217       1.1    Art      21185      21519
       Art1     1      GHI    21246      32.0    Art      21185      21519
       Art1     1      PQR    21278    5003.0    Art      21185      21519

    SECOND DATASTEP
    WOEK.ADLB13 total obs=6

      NAME   SUBJID    LAB    LABDT    LABVAL    NAME    TRTSTDT    TRTENDT

      Bart      2      ABC    20880       2.1    Bart     21217      21519
      Bart      2      GHI    20912      42.0    Bart     21217      21519
      Bart      2      PQR    20943    6003.0    Bart     21217      21519
      Bart      2      ABC    21306       4.1    Bart     21217      21519
      Bart      2      GHI    21338      62.0    Bart     21217      21519
      Bart      2      PQR    21369    8003.0    Bart     21217      21519


    PROCESS
    =======

    /* initial loading */
    options cmplib=(work.functions);

    data ADLB12;
     set lab;
      length name $ 10 trtstdt trtendt 8;
      format trtstdt trtendt date11.;
      call missing(name, trtstdt, trtendt);
      call GetADSL("O", subjid, name, trtstdt, trtendt);
     if name='Art';
    run;
    proc print;
    run;

    data ADLB13 /*(where=(name='Bart'))*/;
     set lab ;
      length name $ 10 trtstdt trtendt 8;
      format trtstdt trtendt date11. ;
      call missing(name, trtstdt, trtendt);
      call GetADSL("O", subjid, name, trtstdt, trtendt);
      if name="Bart";
    run;
    proc print;
    run;


    *                _              _       _
     _ __ ___   __ _| | _____    __| | __ _| |_ __ _
    | '_ ` _ \ / _` | |/ / _ \  / _` |/ _` | __/ _` |
    | | | | | | (_| |   <  __/ | (_| | (_| | || (_| |
    |_| |_| |_|\__,_|_|\_\___|  \__,_|\__,_|\__\__,_|

    ;

    %let _FUNCTIONID_=FUN_GetADSL_VAR_;
    %put *&_FUNCTIONID_.*;


    /* just some fake data */
    data LAB;
    input @1 subjid @3 lab : $ 5. @8 labdt date9. @17 labval best32.;
    format subjid z3. labdt yymmdd10. labval best32.;
    cards;
    1 ABC  2feb2018 1.1
    1 GHI  3mar2018 32
    1 PQR  4apr2018 5003
    2 ABC  2mar2017 2.1
    2 GHI  3apr2017 42
    2 PQR  4may2017 6003
    0 XYZ  29feb2016 .
    3 ABC  2apr2018 3.1
    3 GHI  3may2018 52
    3 PQR  4jun2018 7003
    2 ABC  2may2018 4.1
    2 GHI  3jun2018 62
    2 PQR  4jul2018 8003
    ;
    run;

    data adsl;
     input @1 subjid @3 name : $ 20. @10 trtstdt date9. @20 trtendt date9.;
     format subjid z3. trtstdt trtendt yymmdd10.;
    cards;
    1 Art    1jan2018 31dec2018
    2 Bart   2feb2018 31dec2018
    3 Thomas 3mar2018 31dec2018
    ;
    run;

    *                                      __
     _ __ ___   __ _  ___ _ __ ___  ___   / _| ___ _ __ ___  _ __
    | '_ ` _ \ / _` |/ __| '__/ _ \/ __| | |_ / __| '_ ` _ \| '_ \
    | | | | | | (_| | (__| | | (_) \__ \ |  _| (__| | | | | | |_) |
    |_| |_| |_|\__,_|\___|_|  \___/|___/ |_|  \___|_| |_| |_| .__/
                                                            |_|
    ;

    %put ******%sysfunc(datetime(),datetime25.)******;

    options cmplib=_null_;
    /* inner function */
    proc fcmp outlib=work.functions.hash;
     function GetADSL_inner(mode $, subjid);
      length mode $ 1 subjid 8 name $ 32 trtstdt trtendt 8;
        declare hash ADSL(dataset:'WORK.adslT', duplicate: "r", ordered: "a", hashexp: 20);
         rc = ADSL.DefineKey('subjid');    /* key is numeric */
         rc = ADSL.DefineData(all: "yes"); /* data are BOTH numeric and character */
         rc = ADSL.DefineDone();

        _rc_ = ADSL.check();

        if upcase(mode) = "I" then /* input mode */
        do;
         /* retrieve values from macrovariables created by inner macro */
         name    =  strip(symget("&_FUNCTIONID_.name"   )           );
         trtstdt = inputn(symget("&_FUNCTIONID_.trtstdt"), "best32.");
         trtendt = inputn(symget("&_FUNCTIONID_.trtendt"), "best32.");
         /**/
        _rc_ = ADSL.replace();
        end;

         _rc_ = ADSL.find();

         /* create global macrovariables which will be shared with main function */
         call symputx("&_FUNCTIONID_.name",    name,                    "G");
         call symputx("&_FUNCTIONID_.trtstdt", putn(trtstdt,"best32."), "G");
         call symputx("&_FUNCTIONID_.trtendt", putn(trtendt,"best32."), "G");
         /**/
         return(_rc_);
     endsub;
    run;

    /* macro */
    %put ******%sysfunc(datetime(),datetime25.)******;

    /* macro is just shell/anchor to call inner function */
    %macro callGetADSL(mode=O,subjid=0);
    %local mode subjid;
    %SYSFUNC(GetADSL_inner(&mode.,&subjid.));
    %mend callGetADSL;

    /* outer function i.e subroutine :-) */
    proc fcmp outlib=work.functions.hash;
     SUBROUTINE GetADSL(mode $, subjid, name $, trtstdt, trtendt);
      outargs name, trtstdt, trtendt;
       length str $ 20;
        /* create global macrovariables which will be shared with main function */
        call symputx("&_FUNCTIONID_.name",    name,                    "G");
        call symputx("&_FUNCTIONID_.trtstdt", putn(trtstdt,"best32."), "G");
        call symputx("&_FUNCTIONID_.trtendt", putn(trtendt,"best32."), "G");
        /**/

        str = resolve('%callGetADSL(mode='|| mode ||', subjid=' || put(subjid, best32.) || ');');

        /* retrieve values from macrovariables created by inner macro */
        name    =  strip(symget("&_FUNCTIONID_.name"   )           );
        trtstdt = inputn(symget("&_FUNCTIONID_.trtstdt"), "best32.");
        trtendt = inputn(symget("&_FUNCTIONID_.trtendt"), "best32.");
        /**/

      return;
     endsub;
    run;

    * __       _ _             _       _   _
     / _|_   _| | |  ___  ___ | |_   _| |_(_) ___  _ __
    | |_| | | | | | / __|/ _ \| | | | | __| |/ _ \| '_ \
    |  _| |_| | | | \__ \ (_) | | |_| | |_| | (_) | | | |
    |_|  \__,_|_|_| |___/\___/|_|\__,_|\__|_|\___/|_| |_|

    ;

    * I needed to set these options;
    %utlopts;

    /**###################################################################**/
    /*                                                                     */
    /*  Copyright Bartosz Jablonski, October 2018.                         */
    /*                                                                     */
    /*  Code is free and open source. If you want - you can use it.        */
    /*  But it comes with absolutely no warranty whatsoever.               */
    /*  If you cause any damage or something - it will be your own fault.  */
    /*  You've been warned! You are using it on your own risk.             */
    /*  However, if you decide to use it don't forget to mention author.   */
    /*  Bartosz Jablonski (yabwon@gmail.com)                               */
    /*                                                                     */
    /**###################################################################**/

    /*
    This file is continuation of discusin started in files:
    Function-Hash-Macro-sandwich.sas
    and
    Function-Hash-Macro-sandwich-approach-2.sas

    Read them first.

    If you need help about followin code, contact info:
    Bartosz JabL?oL?ski (yabwon@gmail.com)
    */


    /* this _FUNCTIONID_ macrovariable is to ensure (try to ensure) that macrovariables
       created in the inner function would have "unique" name. In case when someone created
       global mamacrovariable named "subjid" we do not want to overvrite it.
       run it only once at the function definition
    */

    %let _FUNCTIONID_=FUN_GetADSL_VAR_;
    %put *&_FUNCTIONID_.*;


    /* just some fake data */
    data LAB;
    input @1 subjid @3 lab : $ 5. @8 labdt date9. @17 labval best32.;
    format subjid z3. labdt yymmdd10. labval best32.;
    cards;
    1 ABC  2feb2018 1.1
    1 GHI  3mar2018 32
    1 PQR  4apr2018 5003
    2 ABC  2mar2017 2.1
    2 GHI  3apr2017 42
    2 PQR  4may2017 6003
    0 XYZ  29feb2016 .
    3 ABC  2apr2018 3.1
    3 GHI  3may2018 52
    3 PQR  4jun2018 7003
    2 ABC  2may2018 4.1
    2 GHI  3jun2018 62
    2 PQR  4jul2018 8003
    ;
    run;

    data adsl;
     input @1 subjid @3 name : $ 20. @10 trtstdt date9. @20 trtendt date9.;
     format subjid z3. trtstdt trtendt yymmdd10.;
    cards;
    1 Art    1jan2018 31dec2018
    2 Bart   2feb2018 31dec2018
    3 Thomas 3mar2018 31dec2018
    ;
    run;

    %put ******%sysfunc(datetime(),datetime25.)******;

    options cmplib=_null_;

    /* initial loading */
    options cmplib=(work.functions);
    /* inner function */
    proc fcmp outlib=work.functions.hash;
     function GetADSL_inner(mode $, subjid);
      length mode $ 1 subjid 8 name $ 32 trtstdt trtendt 8;
        declare hash ADSL(dataset:'WORK.adslT', duplicate: "r", ordered: "a", hashexp: 20);
         rc = ADSL.DefineKey('subjid');    /* key is numeric */
         rc = ADSL.DefineData(all: "yes"); /* data are BOTH numeric and character */
         rc = ADSL.DefineDone();

        _rc_ = ADSL.check();

        if upcase(mode) = "I" then /* input mode */
        do;
         /* retrieve values from macrovariables created by inner macro */
         name    =  strip(symget("&_FUNCTIONID_.name"   )           );
         trtstdt = inputn(symget("&_FUNCTIONID_.trtstdt"), "best32.");
         trtendt = inputn(symget("&_FUNCTIONID_.trtendt"), "best32.");
         /**/
        _rc_ = ADSL.replace();
        end;

         _rc_ = ADSL.find();

         /* create global macrovariables which will be shared with main function */
         call symputx("&_FUNCTIONID_.name",    name,                    "G");
         call symputx("&_FUNCTIONID_.trtstdt", putn(trtstdt,"best32."), "G");
         call symputx("&_FUNCTIONID_.trtendt", putn(trtendt,"best32."), "G");
         /**/
         return(_rc_);
     endsub;
    run;

    /* macro */
    %put ******%sysfunc(datetime(),datetime25.)******;

    /* macro is just shell/anchor to call inner function */
    %macro callGetADSL(mode=O,subjid=0);
    %local mode subjid;
    %SYSFUNC(GetADSL_inner(&mode.,&subjid.));
    %mend callGetADSL;

    /* outer function i.e subroutine :-) */
    proc fcmp outlib=work.functions.hash;
     SUBROUTINE GetADSL(mode $, subjid, name $, trtstdt, trtendt);
      outargs name, trtstdt, trtendt;
       length str $ 20;
        /* create global macrovariables which will be shared with main function */
        call symputx("&_FUNCTIONID_.name",    name,                    "G");
        call symputx("&_FUNCTIONID_.trtstdt", putn(trtstdt,"best32."), "G");
        call symputx("&_FUNCTIONID_.trtendt", putn(trtendt,"best32."), "G");
        /**/

        str = resolve('%callGetADSL(mode='|| mode ||', subjid=' || put(subjid, best32.) || ');');

        /* retrieve values from macrovariables created by inner macro */
        name    =  strip(symget("&_FUNCTIONID_.name"   )           );
        trtstdt = inputn(symget("&_FUNCTIONID_.trtstdt"), "best32.");
        trtendt = inputn(symget("&_FUNCTIONID_.trtendt"), "best32.");
        /**/

      return;
     endsub;
    run;

    /* initial loading */
    options cmplib=(work.functions);

    data ADLB12;
     set lab;
      length name $ 10 trtstdt trtendt 8;
      format trtstdt trtendt date11.;
      call missing(name, trtstdt, trtendt);
      call GetADSL("O", subjid, name, trtstdt, trtendt);
     if name='Art';
    run;
    proc print;
    run;

    data ADLB13;
     set lab ;
      length name $ 10 trtstdt trtendt 8;
      format trtstdt trtendt date11. ;
      call missing(name, trtstdt, trtendt);
      call GetADSL("O", subjid, name, trtstdt, trtendt);
      if name="Bart";
    run;
    proc print;
    run;

    *_
    | | ___   __ _
    | |/ _ \ / _` |
    | | (_) | (_| |
    |_|\___/ \__, |
             |___/
    ;

    5086  * I needed to set these options;
    5087  %utlopts;
    MLOGIC(UTLOPTS):  Beginning execution.
    MLOGIC(UTLOPTS):  This macro was compiled from the autocall file c:\oto\utlopts.sas
    MPRINT(UTLOPTS):   OPTIONS OBS=MAX FIRSTOBS=1 lrecl=384 NOFMTERR SOURCE SOURCe2 MACROGEN SYMBOLGEN NOTES NOOVP CMDMAC MLOGIC MPRINT MRECALL MERROR NOCENTER DETAI
    NONUMBER FULLSTIMER NODATE DKRICOND=WARN DKROCOND=WARN ;
    MPRINT(UTLOPTS):   run;
    MPRINT(UTLOPTS):  quit;
    MLOGIC(UTLOPTS):  Ending execution.
    5088  /**###################################################################**/
    5089  /*                                                                     */
    5090  /*  Copyright Bartosz Jablonski, October 2018.                         */
    5091  /*                                                                     */
    5092  /*  Code is free and open source. If you want - you can use it.        */
    5093  /*  But it comes with absolutely no warranty whatsoever.               */
    5094  /*  If you cause any damage or something - it will be your own fault.  */
    5095  /*  You've been warned! You are using it on your own risk.             */
    5096  /*  However, if you decide to use it don't forget to mention author.   */
    5097  /*  Bartosz Jablonski (yabwon@gmail.com)                               */
    5098  /*                                                                     */
    5099  /**###################################################################**/
    5100  /*
    5101  This file is continuation of discusin started in files:
    5102  Function-Hash-Macro-sandwich.sas
    5103  and
    5104  Function-Hash-Macro-sandwich-approach-2.sas
    5105  Read them first.
    5106  If you need help about followin code, contact info:
    5107  Bartosz JabL?oL?ski (yabwon@gmail.com)
    5108  */
    5109  /* this _FUNCTIONID_ macrovariable is to ensure (try to ensure) that macrovariables
    5110     created in the inner function would have "unique" name. In case when someone created
    5111     global mamacrovariable named "subjid" we do not want to overvrite it.
    5112     run it only once at the function definition
    5113  */
    5114  %let _FUNCTIONID_=FUN_GetADSL_VAR_;
    5115  %put *&_FUNCTIONID_.*;
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    *FUN_GetADSL_VAR_*
    5116  /* just some fake data */
    5117  data LAB;
    5118  input @1 subjid @3 lab : $ 5. @8 labdt date9. @17 labval best32.;
    5119  format subjid z3. labdt yymmdd10. labval best32.;
    5120  cards;

    NOTE: The data set WORK.LAB has 13 observations and 4 variables.
    NOTE: DATA statement used (Total process time):
          real time           0.01 seconds
          user cpu time       0.01 seconds
          system cpu time     0.00 seconds
          memory              366.46k
          OS Memory           50672.00k
          Timestamp           01/14/2019 04:43:29 PM
          Step Count                        266  Switch Count  0


    5134  ;
    5135  run;
    5136  data adsl;
    5137   input @1 subjid @3 name : $ 20. @10 trtstdt date9. @20 trtendt date9.;
    5138   format subjid z3. trtstdt trtendt yymmdd10.;
    5139  cards;

    NOTE: The data set WORK.ADSL has 3 observations and 4 variables.
    NOTE: DATA statement used (Total process time):
          real time           0.01 seconds
          user cpu time       0.00 seconds
          system cpu time     0.00 seconds
          memory              366.59k
          OS Memory           50672.00k
          Timestamp           01/14/2019 04:43:29 PM
          Step Count                        267  Switch Count  0


    5143  ;
    5144  run;
    5145  %put ******%sysfunc(datetime(),datetime25.)******;
    ******       14JAN2019:16:43:30******
    5146  options cmplib=_null_;
    5147  /* inner function */
    5148  proc fcmp outlib=work.functions.hash;
    5149   function GetADSL_inner(mode $, subjid);
    5150    length mode $ 1 subjid 8 name $ 32 trtstdt trtendt 8;
    5151      declare hash ADSL(dataset:'WORK.adslT', duplicate: "r", ordered: "a", hashexp: 20);
    5152       rc = ADSL.DefineKey('subjid');    /* key is numeric */
    5153       rc = ADSL.DefineData(all: "yes"); /* data are BOTH numeric and character */
    5154       rc = ADSL.DefineDone();
    5155      _rc_ = ADSL.check();
    5156      if upcase(mode) = "I" then /* input mode */
    5157      do;
    5158       /* retrieve values from macrovariables created by inner macro */
    5159       name    =  strip(symget("&_FUNCTIONID_.name"   )           );
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5160       trtstdt = inputn(symget("&_FUNCTIONID_.trtstdt"), "best32.");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5161       trtendt = inputn(symget("&_FUNCTIONID_.trtendt"), "best32.");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5162       /**/
    5163      _rc_ = ADSL.replace();
    5164      end;
    5165       _rc_ = ADSL.find();
    5166       /* create global macrovariables which will be shared with main function */
    5167       call symputx("&_FUNCTIONID_.name",    name,                    "G");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5168       call symputx("&_FUNCTIONID_.trtstdt", putn(trtstdt,"best32."), "G");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5169       call symputx("&_FUNCTIONID_.trtendt", putn(trtendt,"best32."), "G");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5170       /**/
    5171       return(_rc_);
    5172   endsub;
    5173  run;

    NOTE: Function GetADSL_inner saved to work.functions.hash.
    NOTE: PROCEDURE FCMP used (Total process time):
          real time           0.07 seconds
          user cpu time       0.00 seconds
          system cpu time     0.03 seconds
          memory              18367.34k
          OS Memory           67316.00k
          Timestamp           01/14/2019 04:43:29 PM
          Step Count                        268  Switch Count  6


    5174  /* macro */
    5175  %put ******%sysfunc(datetime(),datetime25.)******;
    ******       14JAN2019:16:43:30******
    5176  /* macro is just shell/anchor to call inner function */
    5177  %macro callGetADSL(mode=O,subjid=0);
    5178  %local mode subjid;
    5179  %SYSFUNC(GetADSL_inner(&mode.,&subjid.));
    5180  %mend callGetADSL;
    5181  /* outer function i.e subroutine :-) */
    5182  proc fcmp outlib=work.functions.hash;
    5183   SUBROUTINE GetADSL(mode $, subjid, name $, trtstdt, trtendt);
    5184    outargs name, trtstdt, trtendt;
    5185     length str $ 20;
    5186      /* create global macrovariables which will be shared with main function */
    5187      call symputx("&_FUNCTIONID_.name",    name,                    "G");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5188      call symputx("&_FUNCTIONID_.trtstdt", putn(trtstdt,"best32."), "G");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5189      call symputx("&_FUNCTIONID_.trtendt", putn(trtendt,"best32."), "G");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5190      /**/
    5191      str = resolve('%callGetADSL(mode='|| mode ||', subjid=' || put(subjid, best32.) || ');');
    5192      /* retrieve values from macrovariables created by inner macro */
    5193      name    =  strip(symget("&_FUNCTIONID_.name"   )           );
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5194      trtstdt = inputn(symget("&_FUNCTIONID_.trtstdt"), "best32.");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5195      trtendt = inputn(symget("&_FUNCTIONID_.trtendt"), "best32.");
    SYMBOLGEN:  Macro variable _FUNCTIONID_ resolves to FUN_GetADSL_VAR_
    5196      /**/
    5197    return;
    5198   endsub;
    5199  run;

    NOTE: Function GetADSL saved to work.functions.hash.
    NOTE: PROCEDURE FCMP used (Total process time):
          real time           0.03 seconds
          user cpu time       0.01 seconds
          system cpu time     0.00 seconds
          memory              1203.31k
          OS Memory           50672.00k
          Timestamp           01/14/2019 04:43:29 PM
          Step Count                        269  Switch Count  0


    5200  /* initial loading */
    5201  options cmplib=(work.functions);
    5202  data ADLB12;
    5203   set lab;
    5204    length name $ 10 trtstdt trtendt 8;
    5205    format trtstdt trtendt date11.;
    5206    call missing(name, trtstdt, trtendt);
    5207    call GetADSL("O", subjid, name, trtstdt, trtendt);
    5208   if name='Art';
    5209  run;

    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 1
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 1
    MACROGEN(CALLGETADSL):   ;
    MACROGEN(CALLGETADSL):   function GetADSL_inner(mode $, subjid);
    MACROGEN(CALLGETADSL):   length mode $ 1 subjid 8 name $ 32 trtstdt trtendt 8;
    MACROGEN(CALLGETADSL):   declare hash ADSL(dataset:'WORK.adslT', duplicate: "r", ordered: "a", hashexp: 20);
    MACROGEN(CALLGETADSL):   rc = ADSL.DefineKey('subjid');
    MACROGEN(CALLGETADSL):   rc = ADSL.DefineData(all: "yes");
    MACROGEN(CALLGETADSL):   rc = ADSL.DefineDone();
    MACROGEN(CALLGETADSL):   _rc_ = ADSL.check();
    MACROGEN(CALLGETADSL):   if upcase(mode) = "I" then do;
    MACROGEN(CALLGETADSL):   name = strip(symget("FUN_GetADSL_VAR_name" ) );
    MACROGEN(CALLGETADSL):   trtstdt = inputn(symget("FUN_GetADSL_VAR_trtstdt"), "best32.");
    MACROGEN(CALLGETADSL):   trtendt = inputn(symget("FUN_GetADSL_VAR_trtendt"), "best32.");
    MACROGEN(CALLGETADSL):   _rc_ = ADSL.replace();
    MACROGEN(CALLGETADSL):   end;
    MACROGEN(CALLGETADSL):   _rc_ = ADSL.find();
    MACROGEN(CALLGETADSL):   call symputx("FUN_GetADSL_VAR_name", name, "G");
    MACROGEN(CALLGETADSL):   call symputx("FUN_GetADSL_VAR_trtstdt", putn(trtstdt,"best32."), "G");
    MACROGEN(CALLGETADSL):   call symputx("FUN_GetADSL_VAR_trtendt", putn(trtendt,"best32."), "G");
    MACROGEN(CALLGETADSL):   return(_rc_);
    MACROGEN(CALLGETADSL):   endsub;
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 1
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 1
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 1
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 1
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 0
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 0
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 3
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 3
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 3
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 3
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 3
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 3
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    NOTE: DATA statement used (Total process time):
          real time           0.14 seconds
          user cpu time       0.03 seconds
          system cpu time     0.07 seconds
          memory              2023.09k
          OS Memory           51696.00k
          Timestamp           01/14/2019 04:43:30 PM
          Step Count                        270  Switch Count  8

    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    NOTE: There were 13 observations read from the data set WORK.LAB.
    NOTE: The data set WORK.ADLB12 has 3 observations and 7 variables.

    5210  proc print;
    5211  run;

    NOTE: There were 3 observations read from the data set WORK.ADLB12.
    NOTE: PROCEDURE PRINT used (Total process time):
          real time           0.03 seconds
          user cpu time       0.00 seconds
          system cpu time     0.03 seconds
          memory              349.93k
          OS Memory           51184.00k
          Timestamp           01/14/2019 04:43:30 PM
          Step Count                        271  Switch Count  0


    5212  data ADLB13;
    5213   set lab ;
    5214    length name $ 10 trtstdt trtendt 8;
    5215    format trtstdt trtendt date11. ;
    5216    call missing(name, trtstdt, trtendt);
    5217    call GetADSL("O", subjid, name, trtstdt, trtendt);
    5218    if name="Bart";
    5219  run;

    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 1
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 1
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 1
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 1
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 1
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 1
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 0
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 0
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 3
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 3
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 3
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 3
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 3
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 3
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    NOTE: DATA statement used (Total process time):
          real time           0.04 seconds
          user cpu time       0.01 seconds
          system cpu time     0.01 seconds
          memory              1400.62k
          OS Memory           51184.00k
          Timestamp           01/14/2019 04:43:30 PM
          Step Count                        272  Switch Count  0

    MLOGIC(CALLGETADSL):  Beginning execution.
    MLOGIC(CALLGETADSL):  Parameter MODE has value O
    MLOGIC(CALLGETADSL):  Parameter SUBJID has value 2
    MLOGIC(CALLGETADSL):  %LOCAL  MODE SUBJID
    SYMBOLGEN:  Macro variable MODE resolves to O
    SYMBOLGEN:  Macro variable SUBJID resolves to 2
    MLOGIC(CALLGETADSL):  Ending execution.
    NOTE: There were 13 observations read from the data set WORK.LAB.
    NOTE: The data set WORK.ADLB13 has 6 observations and 7 variables.

    5220  proc print;
    5221  run;

    NOTE: There were 6 observations read from the data set WORK.ADLB13.
    NOTE: PROCEDURE PRINT used (Total process time):
          real time           0.01 seconds
          user cpu time       0.00 seconds
          system cpu time     0.00 seconds
          memory              349.93k
          OS Memory           51184.00k
          Timestamp           01/14/2019 04:43:30 PM
          Step Count                        273  Switch Count  0


    SYMBOLGEN:  Macro variable PGM resolves to utl-sharing-hash-storage-with-two-separate-datasteps-in-the-same-SAS-session
    SYMBOLGEN:  Macro variable PGM resolves to utl-sharing-hash-storage-with-two-separate-datasteps-in-the-same-SAS-session
    SYMBOLGEN:  Macro variable _Q resolves to 57441
    SYMBOLGEN:  Macro variable _Q resolves to 57441


