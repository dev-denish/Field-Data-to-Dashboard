# Field Data to Dashboard

An end-to-end field data collection workflow for VNV. Field teams
capture geotagged points, boundaries, forms, and photos using
**QField**. Records synchronize to a self-hosted cloud server and appear
on the VNV dashboard map without manual file transfers.

> **Current scope:** Phase 1 uses the phone's built-in GNSS (typically
> around 5 m accuracy). Phase 2 adds an RTK receiver and Survey of India
> CORS corrections for centimetre-level positioning.

## Overview

The workflow has four stages:

1.  **Set up:** Office staff create projects, configure layers and
    forms, and assign field personnel in the VNV dashboard.
2.  **Record:** Field personnel collect points, lines, polygons, form
    responses, and photos in QField.
3.  **Synchronize:** QField sends new records when connectivity is
    available. Offline records remain on the phone until synchronization
    is possible.
4.  **Review:** Office staff inspect records, photos, accuracy,
    conflicts, and progress in the dashboard.

``` mermaid
flowchart LR
    A["Office team<br/>VNV dashboard"] --> B["Project and form setup"]
    B --> C["Field phone<br/>QField"]
    C -->|Mobile data / HTTPS| D["QFieldCloud"]
    D --> E[("PostGIS<br/>Spatial records")]
    D --> F[("MinIO<br/>Photo storage")]
    E --> G["VNV dMRV dashboard"]
    F --> G
    H["ntfy<br/>Field notifications"] -.-> C
```

## Technology stack

  --------------------------------------------------------------------------------------
  Component                              Purpose                 Phase
  -------------------------------------- ----------------------- -----------------------
  VNV field dashboard                    Project creation,       1
                                         templates, users,       
                                         notices, review,        
                                         exports, progress       
                                         tracking, and project   
                                         closure                 

  [QField](https://qfield.org/)          Mobile collection of    1
                                         points, lines,          
                                         polygons, forms, and    
                                         photos                  

  [QFieldCloud](https://qfield.cloud/)   Receives field edits    1
                                         and synchronizes        
                                         project data            

  [QGIS](https://qgis.org/)              Builds and publishes    1
                                         project files on the    
                                         server                  

  [PostGIS](https://postgis.net/)        Stores spatial records  1

  [MinIO](https://min.io/)               Stores field photos     1

  [ntfy](https://ntfy.sh/)               Sends office notices to 1
                                         field phones            

  Cloud server                           Hosts the services and  1
                                         dashboard               

  RTK GNSS receiver                      Provides                2
                                         centimetre-level        
                                         positioning when paired 
                                         with correction data    

  Survey of India CORS / NTRIP           Supplies GNSS           2
                                         correction signals      
  --------------------------------------------------------------------------------------

The software components listed above are open source. The cloud server
is rented infrastructure. Phase 2 also requires RTK receiver hardware
and the relevant correction service.

## Architecture

### Phase 1: Phone GNSS

1.  The phone records its location using its built-in GNSS.
2.  QField stores new records locally on the device.
3.  When mobile data is available, QField synchronizes changes with
    QFieldCloud over HTTPS.
4.  QFieldCloud coordinates edits from field personnel.
5.  Spatial records are stored in PostGIS and photos in MinIO.
6.  The VNV dashboard reads the stored data and displays new points and
    photos on the map.

``` mermaid
flowchart TD
    A["QField on phone"] -->|"When online"| B["QFieldCloud"]
    B --> C[("PostGIS database")]
    B --> D[("MinIO photo storage")]
    C --> E["VNV dashboard map"]
    D --> E
    N["ntfy notification service"] -.-> A
```

### Phase 2: RTK positioning

Phase 2 adds an RTK receiver connected to the phone over Bluetooth.
Survey of India CORS correction data is delivered using NTRIP. The
application and server workflow remain the same; the positioning
equipment changes.

``` mermaid
flowchart LR
    A["Survey of India CORS"] -->|"NTRIP corrections"| B["Phone / QField"]
    B <-->|Bluetooth| C["RTK receiver"]
    C --> D["Centimetre-level position"]
    D --> B
```

## Accuracy expectations

  ------------------------------------------------------------------------
  Phase            Equipment             Typical accuracy Intended use
                                      stated in the guide 
  ---------------- ---------------- --------------------- ----------------
  Phase 1          Phone GNSS                   About 5 m Geotagged
                                                          photos, site
                                                          visits,
                                                          observations,
                                                          approximate plot
                                                          outlines

  Phase 2          Phone + RTK                About ±3 cm Plot boundaries
                   receiver +                             and areas,
                   Survey of India                        sample points,
                   corrections                            and measurements
                                                          used for carbon
                                                          verification
  ------------------------------------------------------------------------

**Important:** Phase 1 plot boundaries are approximate and should be
labelled accordingly in the dashboard. The guide illustrates why this
matters: a 100 m × 100 m plot recorded 5 m too wide on every side
becomes 110 m × 110 m, or 1.21 ha instead of 1 ha.

For better phone positioning: - Work in open sky, away from tall trees,
walls, and vehicles. - Allow around 20--30 seconds for the GPS accuracy
reading to settle. - Average multiple readings for important points. -
Use the dashboard's accuracy review to flag records outside the
configured limit. - Retain the recorded accuracy with each feature.

## Field team workflow

### First-time phone setup

1.  Install QField and ntfy.
2.  Sign in to QField using the credentials supplied by the office.
3.  Download the assigned project and its offline maps.
4.  Subscribe to the personal ntfy channel supplied by the office.
5.  Allow notifications and disable battery optimization for ntfy.

### Before visiting a site

-   Charge the phone and carry a power bank.
-   Enable Location in high-accuracy mode.
-   Open the assigned QField project and confirm the current position
    appears on the map.
-   Check ntfy for office notices.
-   Confirm that previous records have finished synchronizing.

### Recording a point or plot

1.  Stand at the location to be recorded, preferably under open sky.
2.  Wait until the GPS indicator is green and the accuracy value has
    settled.
3.  For important points, use averaging where appropriate.
4.  For a plot, walk its boundary and record each corner.
5.  Complete the form using the configured dropdowns and fields.
6.  Take photos from inside the QField form so they remain attached to
    the feature.
7.  Save the record. It is stored on the phone even if there is no
    network connection.

### During and after fieldwork

-   Synchronization happens when connectivity is available.
-   If there is no signal, continue collecting data. Pending records
    remain on the phone and can synchronize later.
-   At the end of the day, return to coverage, open QField, and wait
    until no records are waiting to send.
-   Keep the phone and power bank charged.

### Important precautions

-   Do not uninstall QField or clear its data while records are still
    waiting to synchronize.
-   Do not take survey photos using the normal camera app if they need
    to be linked to a feature.
-   Do not record while the GPS indicator is red.
-   Do not ignore a request to synchronize before a form update.
-   Do not update QField or ntfy in the field unless the office has
    tested and approved the update.

## Office dashboard workflow

The office team manages the project from the VNV dashboard. The guide's
intended workflow is:

  ------------------------------------------------------------------------
                          Step Office action         System behaviour
  ---------------------------- --------------------- ---------------------
                             1 Create a project,     Server builds and
                               enter its details,    publishes the QField
                               and upload a boundary project
                               in KML, shapefile, or 
                               GeoJSON format        

                             2 Select layer          Database tables and
                               templates and         mobile forms are
                               configure form fields created to match

                             3 Add field personnel   Logins are created
                               and assign projects   and welcome notices
                                                     are sent

                             4 Update a form during  Safety checks,
                               a project             notices, readiness
                                                     tracking, and
                                                     publishing are
                                                     managed

                             5 Review records as     Review status is
                               verified, needs       stored and re-visit
                               re-visit, or          notices can be sent
                               rejected; add         
                               comments              

                             6 Export Excel/CSV,     Files are generated
                               KML, shapefile, or    from the database
                               GeoJSON with photo    
                               links                 

                             7 Track points per      Progress counts
                               person per day,       update as records
                               photos received, and  arrive
                               remaining work        

                             8 Close or archive a    Field collection is
                               project               stopped and project
                                                     data becomes
                                                     read-only and remains
                                                     exportable
  ------------------------------------------------------------------------

### Standard layer templates

The guide proposes these initial VNV templates:

-   Farm plots
-   Trees
-   Soil samples
-   Water points

Forms can include text, number, dropdown, and photo fields. Standard
metadata includes collector, timestamp, GPS accuracy, and form version.

### Roles

  Role              Permissions described in the guide
  ----------------- -------------------------------------------------
  Admin             Full project and dashboard administration
  Office reviewer   Review records, export data, and track progress
  Field person      Collect field data in QField on the phone

## Safe form updates during an active project

A form update must not break records already stored on field phones.

Allowed changes described in the guide: - Add a dropdown choice. - Add a
new optional field. - Change field labels or help text.

Blocked changes described in the guide: - Rename an existing field. -
Delete an existing field. - Change an existing field's data type.

The update process is:

1.  Office staff draft the form change in the dashboard.
2.  The dashboard runs a safety check.
3.  Field personnel receive a heads-up notice and synchronize their
    phones.
4.  Each person confirms readiness.
5.  The office publishes when everyone is ready or the deadline passes,
    with a warning for anyone who has not confirmed.
6.  Field personnel receive an update notice and synchronize to download
    the new form.
7.  The dashboard tracks the form version on incoming records and
    reminds users who are still using an older version.

The previous form version is retained so the office can roll back if
required.

## Dashboard review checklist

The office should review field submissions each evening:

-   **Map:** Check points, lines, and plots on the map; open each
    feature to inspect form responses and photos.
-   **Accuracy:** Review records that exceed the configured accuracy
    limit.
-   **Counts:** Compare dashboard point and photo counts with field
    reports.
-   **Conflicts:** Resolve records where multiple people edited the same
    plot.
-   **Re-visits:** Add doubtful or flagged records to the next day's
    field list.

## Notifications

The system uses ntfy, hosted on the VNV server, to send notices to field
phones.

Examples of notices: - **Please sync before...** Connect to the network,
open QField, synchronize, and confirm the notice. - **New form ready**
Open QField and synchronize to receive the updated form. - **Please
re-visit...** Check the specified point again. - **Project closed** Stop
collecting records for that project.

## Troubleshooting

  -----------------------------------------------------------------------
  Problem                             What to check or do
  ----------------------------------- -----------------------------------
  No mobile signal                    Continue recording. Pending records
                                      should synchronize when the phone
                                      returns to coverage.

  GPS indicator stays red             Wait 1--2 minutes, move to open
                                      sky, and confirm Location is
                                      enabled in high-accuracy mode.

  Accuracy value is high (for         Wait longer, move to a clearer
  example, 15 m or more)              location, or use averaging. Record
                                      a note if the observation must
                                      still be captured.

  ntfy notices are late or missing    Check the subscribed channel and
                                      disable battery optimization for
                                      ntfy. The office should follow up
                                      with people who have not responded.

  Old-form reminder appears           Open QField when connected and
                                      synchronize.

  Photos are missing from the         Keep QField open on a connection
  dashboard                           and allow synchronization to
                                      finish. Inform the office before
                                      deleting anything.

  Two people edited the same plot     The office reviews the conflict and
                                      chooses the correct version.

  Cloud server is unavailable         Continue field collection. Pending
                                      records remain on the phone until
                                      the server is available.

  App update causes a problem         Do not update in the field. The
                                      office should test updates before
                                      asking field personnel to install
                                      them.
  -----------------------------------------------------------------------

## Deployment requirements and first-field-day checklist

### Decisions required

-   Approve the cloud server provider and account.
-   Allocate a fixed public IP address and web address.
-   Assign office admin and reviewer roles.

### Phase 1 setup

-   [ ] Provision a cloud server. The guide suggests a Mumbai-region
    server with approximately 4 GB RAM (8 GB recommended), 20 GB disk,
    and additional space for photos.
-   [ ] Install QFieldCloud, PostGIS, MinIO, ntfy, and the VNV field
    dashboard.
-   [ ] Configure HTTPS and daily backups.
-   [ ] Build dashboard functions for projects, form templates, people,
    notices, review, export, progress, and project closure.
-   [ ] Create the Farm plots, Trees, Soil samples, and Water points
    templates.
-   [ ] Create the first project and label Phase 1 plot boundaries as
    approximate.
-   [ ] Test that three points with photos appear on the dashboard.
-   [ ] Test the complete form-update notice workflow on two phones.
-   [ ] Train field personnel using the field procedure in this README.

### Later work

-   [ ] Add dashboard-generated PDF reports with photos.
-   [ ] Select and procure RTK receiver hardware.
-   [ ] Register for Survey of India CORS and confirm coverage at
    project sites.
-   [ ] Test positioning against a known survey mark.
-   [ ] Re-survey Phase 1 plot boundaries that will be used in carbon
    claims.

## Why QField?

The guide selects QField because it is open source, supports offline
work, can connect to RTK receivers without an additional helper app, and
synchronizes with a server controlled by VNV.

### Comparison with SW Maps

The guide compares QField with SW Maps, the app initially considered:

  -----------------------------------------------------------------------
  Capability              QField                  SW Maps, as described
                                                  in the guide
  ----------------------- ----------------------- -----------------------
  Points, lines, and      Supported               Supported
  areas                                           

  Forms and photos        Supported               Supported

  Offline maps            Supported               Supported

  Automatic               Supported through       Manual file upload to
  synchronization         QFieldCloud             an FTP server

  Accuracy gate and       Supported               The guide does not
  averaging                                       describe equivalent
                                                  workflow features

  Raw GNSS data logging   The guide identifies    Supported
                          this as a limitation    

  Create new forms in the Forms are prepared in   Supported
  field                   the office              
  -----------------------------------------------------------------------

The guide notes that QField's office-prepared forms help keep data
consistent across the team. Raw GNSS logging may matter in Phase 2 where
correction coverage is unavailable and a receiver with internal logging
is used.

### Other options considered

  -----------------------------------------------------------------------
  Option                  Strength noted in the   Reason it was not the
                          guide                   first choice
  ----------------------- ----------------------- -----------------------
  Mergin Maps             Simple QGIS-based field Fewer survey features
                          app with                than QField
                          synchronization         

  ODK Collect + ODK       Established             Weaker for drawing
  Central                 questionnaire surveys   plots on a map; RTK
                                                  requires a separate
                                                  helper app

  KoboToolbox             Forms; already familiar Similar limitations to
                          at VNV                  ODK for this workflow
  -----------------------------------------------------------------------

## References

The source guide lists these references for implementation and further
reading:

-   [QField GNSS documentation](https://docs.qfield.org/)
-   [QField releases](https://github.com/opengisch/QField/releases)
-   [QFieldCloud self-hosting documentation](https://qfield.cloud/)
-   [QGIS documentation](https://qgis.org/)
-   [PostGIS documentation](https://postgis.net/documentation/)
-   [MinIO documentation](https://min.io/docs/)
-   [ntfy documentation](https://docs.ntfy.sh/)
-   [Survey of India](https://surveyofindia.gov.in/)

> **Source note:** This README is based on the supplied *Field Data to
> Dashboard: End-to-End Guide* (dated 5 November 2026). It describes the
> proposed workflow and requirements in that guide; it does not claim
> that the services have already been deployed or tested in production.
