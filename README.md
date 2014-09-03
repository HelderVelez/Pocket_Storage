##**Pocket_Storage**
**Pocket_BI** is a PL/SQL Business Intelligence (**BI**) framework with the components: 

* [Pocket_ETL](https://github.com/HelderVelez/Pocket_ETL) 
* **Pocket_Storage**, here documented
* Pocket_Reports, to be documented later

The Pocket_BI framework has the ability to update and be updated from a **Couch_DB** database to provide an exterior backup and a **Web frontend**.   

**Pocket_Storage in short:**
:  It provides a **Generic Storage** for all the textual information, via an API.  There is no need to create any other table and it offers a controlled fade away of short-term information, resulting in a slower growth of the storage. 

[Pocket_ETL](https://github.com/HelderVelez/Pocket_ETL) address the <b>E</b>xtraction phase of the **ETL** .  It simplifies the extraction of data from Oracle sources to populate the *stagging area* and can also launch jobs. 

The **Pocket_Storage** addresses the storage concepts needed in the <b>T</b>ransformation and <b>L</b>oading phases, bearing in mind that it will have to support the easy construction of *reports*. 
Departing from the structure of a **Generic Business Report**, that presents some *views of a process*, we will find the best structure of the **Generic Storage** of data.  
An example of a  *process*:  The document handling process in the institution and the *views* of that process could be: the details of the documents initiated and the ones in transit and the concluded ones in each department by subject and clerk on a given period.   
The application **Pocket_Reports**, to be documented later, will detail the automated creation and publishing of reports based on Pocket_Storage. 
 
 
- [**Pocket_Storage**](#pocketstorage)
    - [**Concepts**](#concepts)
        - [**Generic Business Report**](#generic-business-report)
        - [**Generic Storage**](#generic-storage)
    - [**API**](#api)
        - [Packages](#Packages)
        - [Main <b>P</b>rocedures/<b>F</b>unctions](#main-bpbroceduresbfbunctions)
        - [less used, auxiliary or private](#less-used-auxiliary-or-private)
    - [**Tables**](#tables)
        - [**Definitions**](#definitions)
            - [**Users,Institutions,Departments**](#usersinstitutionsdepartments)
            - [**Processes,Views**](#processesviews)
        - [**DATA Store**](#data-store-)

###**Concepts**
####     **Generic Business Report**
* **Report columns**
    * Title
    * Institution and logo
    * date of emission
    * Columns with titles:
      *  *Dimensions* or classifiers, in the left side
      *  *Measures*   with units and date, in the right side
    * optional calculated elements as pagination, totals, partials , ...
 

####**Generic Storage**
* **Definitions**
    * Users (Users table)
    * Institution: ( Inst table)
        * Department:  ( IDept table )
            * Processes:  ( Proc table )
                * Views:  ( PViews table) - enumerate the classifiers, the units and meaning of the measures and an optional period of persistence of the information under that classifier [^period_val].

* **DATA Store**  
Tables **Imgs/Deim** with dated snapshots of information (*Images*), the *facts in BI*, plus table **Imab** for undated data
    * imgs_id - snapshot key
    * view_id - link to definitions to know *what* is recorded
    * ref_date
    *  Detail of Image (table Deim)
        * 8 classifiers of length 100
        * 4 measures (2x2)
            * count
            * value
            * count'
            * value'
 
Large ensembles of data have **errors**. The majority of them are hidden within the data but some spikes unfortunately are visible in the report. The prime(') measures can be used to **store  the corrected values**, after  manual or automatic purification as done in the package **KBeau**(tifier) or, they may store raw measures as well. The BI commercial applications ignore this important issue. Somewhat naively they use distinct tables to store the *facts* of each process/view.

The tables **Imgs/Deim - Images** provide a unified store of snapshots (facts) with dated dimensions and measures. A view_key links to the Definitions information.  
The table **Imab (open images)** is used to store the **undated** information or still evolving. It mimics the Deim table structure with a *view key* added. This attribute together with the 8 classifiers are the primary key. **Closing** this image with a date moves the data from Imab to Imgs/Deim. The table *Imab* can be used as a notebook, a logbook, as auxiliary tables or to aggregate data (an insertion thru the API will add to the measures on the corresponding classifiers).
 
With the help of the *Definitions* we are able to assign the correct titles to the report's columns. 
The parallelism between the structure of a *generic report* and the Image's Storage  allow us to implement the reports in a very general and easy way.
 

 
### **API**
#### Packages

>|PACKAGE|Purpose|reserved View_Id|Obs|
|:--|:--|--:|:--:|
|KHOUS|API|2|main API code
|KBEAU|Beautifier|1|optional,not documented here
|KDOC_SRCS|schema documentation|3,4|idem
|KCOUCH|CouchDB||idem
|KDOC|example to institution documentation process|108..114|idem


#### Main <b>P</b>rocedures/<b>F</b>unctions

 **v_img record**  is the data structure to hold the data passed into the API.
 
>* **v_img** IS RECORD 
  * VIEW_ID View_Id,
  *   IA_P1..P8 , 8 classifiers, varchar2 (100)
  * IA_UN   varchar2 (3), unit of measure
  * IA_C    integer, measure, default 1
  * IA_S    number, measure, default 0
  * IA_C_b  integer, corrected measure, default IA_C or 1
  * IA_S_b  number, corrected measure, default IA_S or 0 
 
 
>|P/F|Name|Description|
|:--|:--|:--|
|P|**p_imgini**(**View_Id** IN IMAB.VIEW_ID%TYPE);|initiate data gather, *imab* will be used to aggregate data|
|P|**p_imgnew_rec**(<br>**img_rec** IN v_img%type, <br> **pcommit** IN boolean default true);|insert/update an image record into *imab*, view_key and the classifiers are the key where the measures are added,<br>  pcommit = true -> commit each record|
|P|**p_imgnew_rec_no_commits**(<br>*img_rec* IN v_img%type);|equal to p_imgnew_rec but without commits|
|P| **p_imgclose**(<br>*View_Id*, <br> *img_date* varchar2(10) *ref_date*,<br> *data_cria* IN creation date, <br> *puserKey* in |data is moved from *imab* to *imgs/deim* using View_Id and ref_date |
|F|**p_img_direct_open** (<br>*View_Id*   IN IMAB.VIEW_ID%TYPE,<br> *img_date*  IN IMGS.IMGS_daim%TYPE,<br>                             *data_cria* IN IMGS.IMGS_dacr%TYPE)|initiate data gather  directly in Imgs/Deim, without using Imab <b> returns: IMGS.IMGS_ID%TYPE
|F| **p_img_direct_new_rec**( <br>*imgKey*    IN DEIM.IMGS_ID%TYPE,<br>*deim_chpa* IN DEIM.DI_ID1%TYPE,<br>*commitYN*  IN boolean,<br>*img_rec*   IN v_img%TYPE)| inserts a record directly in Imgs/Deim <br> returns: **boolean**
|P|**p_img_direct_close**(<br>*imgKey* IN IMGS.IMGS_ID%TYPE)| ends the data gather for imgKey
 
The preferred and faster use is the img_direct path:  open, repeated new_rec and close, that is the case that when the SQL already has the data aggregated by use of Group By. Otherwise use p_imgini, repeated p_imgnew_rec and p_imgclose and the table Imab will do the aggregations. 

  
#### less used, auxiliary or private
>|Procedure<br>Function|Name and parameters|Description|
|:--|:--|:--|
|P|**p_TLOG**(proc ,msg ,PUSER_ID );|write msg in the log|
|P|**p_imgsng_img**(img_rec  IN v_img, <br> img_date varchar2(10) ref_date, <br> PUserKey;|insert an image with a single record
 
 
### **Tables**
#### **Definitions**
##### **Users,Institutions,Departments**

>|Users|Users||||
|-----|-----|-----|--------|-------||
||user_id|number()|not null|user KEY
||idept_id|number()||
||user_sigla|varchar2(12)|not null|user SIGLA
||user_name|varchar2()|not null|
||user_email|varchar2()||
||**special fields**||CouchDB related||
||type|varchar2(7)||
||time_stamp|varchar2()||when updated
||rev|varchar2()||
||id|varchar2()||
||tags|varchar2(19)||
|||||
|**Inst**|**Institutions**
||**inst_id**|number()|not null|key
||inst_name|varchar2()|not null|name
||inst_country|varchar2()||
||inst_address|varchar2()||
||inst_sigla|varchar2()||
||**special fields**||CouchDB related||
||type|varchar2(6)||
||time_stamp|varchar2()||when updated
||**user_id**|number(1)||INST admin user
||rev|varchar2()||
||id|varchar2()||
||tags|varchar2(17)||
|||||
>|**Idept**|**Departments**||||
||**idept_id**|number()|not null|key
||**inst_id**|number()|not null|key of institution
||dept_name|varchar2()|not null|name
||**special fields**||CouchDB related||
||type|varchar2(7)||
||time_stamp|varchar2()||when updated
||user_id|number(1)||dept admin user
||rev|varchar2()||
||id|varchar2()||
||tags|varchar2(17)||
|||||

##### **Processes,Views**

>|**Proc**|**Processes**||||
|:-----|:-----|:-----|:-----|:-------|:--|
||**proc_id**|number()|not null|PK
||proc_sigl|varchar2()|not null|
||inst_id|number()||
||proc_desc|varchar2()|not null|
||idept_id|number(1)||
||proc_stat|varchar2()||
||proc_visi|varchar2()||
||proc_obs|varchar2()||
||proc_suby|varchar2()||
||**special fields**||CouchDB related||
||type|varchar2(6)||
||time_stamp|varchar2()||when updated
||user_id|number(1)||proc  admin user
||rev|varchar2()||cdb _rev
||id|varchar2()||cdb _id
||tags|varchar2(48)||
||status|varchar2(5)||OK, ERROR, TO_VALIDATE,
||msg|varchar2()||
||**triggers**|||
||Gproc
||||
|**Pviews**|**Views**||||
||**view_id**|number()|not null|View Key|
||**proc_id**|number()||Process Key|
||view_sigl|varchar2()|not null|View Acronym|
||view_desc|varchar2()||View Description|
||view_upfn|varchar2()|not null|Updated by Function|
||view_foda|varchar2()|not null|Reference Date Format|
||view_coup|varchar2()||Call By on Update|
||view_couf|varchar2()||Call By on Freeze|
||view_peup|varchar2()||Update Frequency|
||view_pefr|varchar2()||Freeze Frequency|
||view_upfr|varchar2()||Update Then Freeze|
||view_srda|varchar2()||Sql Source of Reference Date|
||view_deco|varchar2()||Counter Description|
||view_deso|varchar2()||Value Description|
||view_un|varchar2()||Value Unit Symbol|
||view_dep1..8|varchar2()||Par1..Par8 Description
||view_hep1..8|varchar2()||Par1..Par8 Header
||view_dtp1..8|number()||Par1..Par8 Months until Drop
||view_obse|varchar2()||Observations
||**special fields**||CouchDB related||
||type|varchar2(8)||
||time_stamp|varchar2()||when updated
||user_id|number(1)||user
||rev|varchar2()||cdb _rev
||id|varchar2()||cdb _id
||tags|varchar2(62)||tags
||||
 
 
#### **DATA Store**

>|Imgs|Images||||
|-----|-----|-----|--------|-------|
||**imgs_id**|number()|not null|Key| 
||**view_id**|number()||
||imgs_daim|varchar2()|not null|reference date of the snapshot|
||imgs_dacr|date()|not null|creation date begin|
||imgs_un|varchar2()|units|
||imgs_dacf|date()||creation date end|
||**special fields**||CouchDB related||
||imgs_couch|varchar2()||
||type|varchar2(6)||
||time_stamp|varchar2()||when updated|
||user_id|number(2)|not null|user id|
||rev|varchar2()||
||id|varchar2()||
||tags|varchar2(33)||
||||||
|**Deim**|**Image Detail**|||||
||**imgs_id**|number()|not null|Image key|
||di_id1|number()|not null|deim chpa|
||di_p1..p8|varchar2()||dimensions 1..8|
||di_c|number()|not null|counter1|
||di_s|number()|not null|total1|
||di_c_b|number()||counter2|
||di_s_b|number()||total2|
||||||
|**Imab**|**Open Image**||||
||**view_id**|number()|not null|VIEW KEY
||ia_p1..p8|varchar2()|not null|dimensions 1..8
||ia_c|number()|not null|counter1
||ia_s|number()||total1
||ia_un|varchar2()||units
||ia_c_b|number()||counter2
||ia_s_b|number()||total2


[**MIT** License](http://opensource.org/licenses/mit-license.php)  
&copy;  2013, Helder Velez.  
 
> Written with [StackEdit](https://stackedit.io/).
  
 [^period_val]: The **temporal scope** of some attributes are limited to the immediate management and others must be persistent in the long term. The rearrangement of the information allows a smaller storage growth. This **controlled *degradé*** of the details  is not present in commercial BI applications.
