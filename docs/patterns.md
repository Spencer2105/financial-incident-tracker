# Patterns used in this project

## 1. Model (Layer 1)

Plain objects with a factory function. No classes — Apps Script's V8 doesn't handle `class` inheritance across files well when all files share the same global scope.

```javascript
// 1_IncidentModel.js
function IncidentModel(data) {
  return {
    id:           data.id           || null,
    area:         data.area         || '',
    category:     data.category     || '',   // 'penalty' | 'loss'
    amount:       data.amount       || 0,
    stage:        data.stage        || 'OPEN',
    responsibleId: data.responsibleId || null,
    createdAt:    data.createdAt    || new Date(),
    updatedAt:    data.updatedAt    || new Date(),
  };
}
```

## 2. Repository (Layer 3)

Treats a Google Sheet tab as a table. Each row maps to a model. Column positions are defined as constants so a column rename in the sheet only requires one code change.

```javascript
// 3_IncidentRepository.js
var INCIDENT_SHEET = 'Incidents';
var COL = { ID: 1, AREA: 2, CATEGORY: 3, AMOUNT: 4, STAGE: 5, RESPONSIBLE: 6, CREATED: 7, UPDATED: 8 };

function findIncidentById(id) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(INCIDENT_SHEET);
  var data  = sheet.getDataRange().getValues();
  for (var i = 1; i < data.length; i++) {        // skip header row
    if (data[i][COL.ID - 1] === id) {
      return IncidentModel({
        id:           data[i][COL.ID - 1],
        area:         data[i][COL.AREA - 1],
        category:     data[i][COL.CATEGORY - 1],
        amount:       data[i][COL.AMOUNT - 1],
        stage:        data[i][COL.STAGE - 1],
        responsibleId: data[i][COL.RESPONSIBLE - 1],
        createdAt:    data[i][COL.CREATED - 1],
        updatedAt:    data[i][COL.UPDATED - 1],
      });
    }
  }
  return null;
}
```

## 3. Service — stage validation (Layer 4)

The workflow has a fixed state machine: `OPEN → FINANCE_REVIEW → LEGAL_REVIEW → CLOSED`. Each transition has required fields. The service enforces this before the repository writes.

```javascript
// 4_IncidentService.js
var VALID_TRANSITIONS = {
  'OPEN':           ['FINANCE_REVIEW'],
  'FINANCE_REVIEW': ['LEGAL_REVIEW', 'OPEN'],
  'LEGAL_REVIEW':   ['CLOSED', 'FINANCE_REVIEW'],
  'CLOSED':         [],
};

function advanceStage(incidentId, targetStage, payload) {
  var incident = findIncidentById(incidentId);
  if (!incident) throw new Error('Incident not found: ' + incidentId);

  var allowed = VALID_TRANSITIONS[incident.stage] || [];
  if (allowed.indexOf(targetStage) === -1) {
    throw new Error('Invalid transition: ' + incident.stage + ' → ' + targetStage);
  }

  validateRequiredFields(targetStage, payload);   // throws if missing

  incident.stage     = targetStage;
  incident.updatedAt = new Date();
  updateIncident(incident);   // repository call

  if (targetStage === 'LEGAL_REVIEW') {
    sendStageNotification(incident, 'Legal team');
  }
  return incident;
}
```

## 4. HTML include pattern (Layer 6)

Apps Script's `HtmlService` can't import JS/CSS files directly. The pattern is to store reusable fragments as `.html` files and include them server-side.

```javascript
// 8_FuncionesInclude.js
function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}
```

```html
<!-- 5_PaginaDashboard.html -->
<!DOCTYPE html>
<html>
  <head>
    <?!= include('6_EstilosBBVA') ?>
  </head>
  <body>
    <?!= include('6_HeaderBBVA') ?>
    <?!= include('6_MenuNavegacion') ?>

    <div id="content">
      <!-- dashboard content -->
    </div>

    <?!= include('6_ScriptDashboard') ?>
  </body>
</html>
```

## 5. Controller routing (Layer 7)

Apps Script web apps have a single entry point: `doGet(e)`. Routing is done by reading a `page` query parameter.

```javascript
// 7_EntryPoints.js
function doGet(e) {
  var page = e.parameter.page || 'dashboard';

  var pages = {
    'dashboard':  '5_PaginaDashboard',
    'incidents':  '5_PaginaDatosGenerales1',
    'finance':    '5_PaginaPenFinanzas1',
    'legal':      '5_PaginaQueLegal1',
  };

  var template = pages[page];
  if (!template) {
    return HtmlService.createHtmlOutputFromFile('5_PaginaAccesoDenegado');
  }

  return HtmlService.createTemplateFromFile(template)
    .evaluate()
    .setTitle('Incident Tracker')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}
```
