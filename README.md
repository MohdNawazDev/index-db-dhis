# Step-by-Step IndexedDB Implementation Guide

## Phase 1: Database Structure Setup

### Step 1: Install Dependencies
```bash
npm install dexie
```

### Step 2: Create Database Configuration (`src/indexDB/db.js`)

```javascript
import Dexie from "dexie";

// Import all manager modules
import * as me from "./MeManager";
import * as organisationUnit from "./OrganisationUnitManager";
import * as organisationsUnitLevel from "./OrganisationUnitLevelManager";
import * as enrollment from "./EnrollmentManager";
import * as trackedEntity from "./TrackedEntityManager";
import * as program from "./ProgramManager";
import * as optionSet from "./OptionSetManager";
import * as event from "./EventManager";
import * as importFile from "./ImportFileManager";

// Create Dexie database instance
export const db = new Dexie("FI_Offline");

// Define database schema
db.version(2).stores({
  [me.TABLE_NAME]: me.TABLE_FIELDS,
  [organisationUnit.TABLE_NAME]: organisationUnit.TABLE_FIELDS,
  [organisationsUnitLevel.TABLE_NAME]: organisationsUnitLevel.TABLE_FIELDS,
  [enrollment.TABLE_NAME]: enrollment.TABLE_FIELDS,
  [trackedEntity.TABLE_NAME]: trackedEntity.TABLE_FIELDS,
  [program.TABLE_NAME]: program.TABLE_FIELDS,
  [optionSet.TABLE_NAME]: optionSet.TABLE_FIELDS,
  [event.TABLE_NAME]: event.TABLE_FIELDS,
  [importFile.TABLE_NAME]: importFile.TABLE_FIELDS,
});

export default db;
```

### Step 3: Create MeManager (`src/indexDB/MeManager/index.js`)

```javascript
export const TABLE_FIELDS = 
  "id, lastUpdated, created, name, displayName, externalAccess, surname, " +
  "lastCheckedInterpretations, firstName, favorite, access, userCredentials, " +
  "sharing, settings, favorites, teiSearchOrganisationUnits, translations, " +
  "organisationUnits, dataViewOrganisationUnits, userGroupAccesses, " +
  "attributeValues, userGroups, userAccesses, authorities, programs, dataSets";

export const TABLE_NAME = "me";

export * from "./MeManager";
```

### Step 4: Create MeManager Implementation (`src/indexDB/MeManager/MeManager.js`)

```javascript
import db from "../db";
import { metadataApi } from "@/api";
import { TABLE_NAME } from ".";

export const pull = async () => {
  try {
    console.log("Downloading user metadata...");
    await db[TABLE_NAME].clear();

    const me = await metadataApi.getMe();
    
    if (me) {
      await addMe([me]);
      console.log("User metadata stored successfully");
    }
  } catch (error) {
    console.error(`Failed to download user metadata:`, error);
    throw error;
  }
};

export const addMe = async (me) => {
  try {
    await db[TABLE_NAME].bulkPut(me);
  } catch (error) {
    console.error(`Failed to store user metadata:`, error);
    throw error;
  }
};

export const getMe = async () => {
  try {
    const me = await db[TABLE_NAME].toArray();
    return me?.length ? me[0] : null;
  } catch (error) {
    console.error(`Failed to get user data:`, error);
    return null;
  }
};
```

### Step 5: Create OrganisationUnitManager (`src/indexDB/OrganisationUnitManager/index.js`)

```javascript
export const TABLE_FIELDS = "id, code, displayName, level, path, withinUserHierarchy";
export const TABLE_NAME = "organisationUnit";

export * from "./OrganisationUnitManager";
```

### Step 6: Create OrganisationUnitManager Implementation (`src/indexDB/OrganisationUnitManager/OrganisationUnitManager.js`)

```javascript
import db from "../db";
import { get, groupBy } from "lodash";
import { metadataApi } from "@/api";
import { TABLE_NAME } from ".";
import * as meManager from "@/indexDB/MeManager/MeManager";

export const pull = async () => {
  try {
    console.log("Downloading organization units...");
    await db[TABLE_NAME].clear();
    
    // Get all organization units within user hierarchy
    const orgs = await metadataApi.get(`/api/organisationUnits`, {}, [
      "withinUserHierarchy=true&paging=false&fields=id,code,path,displayName,level,parent,translations",
    ]);

    if (orgs.organisationUnits && orgs.organisationUnits.length > 0) {
      await addOrgs(orgs.organisationUnits);
    }

    // Get user's specific organization units
    const orgsByUser = await metadataApi.getUserOrgUnits();
    if (orgsByUser.organisationUnits && orgsByUser.organisationUnits.length > 0) {
      const addWithinUserHierarchy = orgsByUser.organisationUnits.map((org) => {
        org.withinUserHierarchy = 1;
        return org;
      });
      await addOrgs(addWithinUserHierarchy);
    }
    
    console.log("Organization units stored successfully");
  } catch (error) {
    console.error(`Failed to download organization units:`, error);
    throw error;
  }
};

export const addOrgs = async (orgUnits) => {
  try {
    // Group by level for proper hierarchy building
    const byLevel = groupBy(orgUnits, (org) => org.level);
    const levels = Object.keys(byLevel).sort();
    const persisted = {};

    for (const level of levels) {
      const objects = byLevel[level];
      await db[TABLE_NAME].bulkPut(objects);
      
      objects.forEach((org) => {
        persisted[org.id] = org;
      });
    }
  } catch (error) {
    console.error(`Failed to store organization units:`, error);
    throw error;
  }
};

export const getUserOrgs = async () => {
  try {
    const orgs = await db[TABLE_NAME].where("withinUserHierarchy").equals(1).toArray();

    // Add children to each org
    for (const org of orgs) {
      const childrenOrgs = await db[TABLE_NAME]
        .where("path")
        .startsWith(org.path + "/")
        .toArray();
      org.children = childrenOrgs;
    }

    return { organisationUnits: orgs };
  } catch (error) {
    console.error(`Failed to get user organization units:`, error);
    return { organisationUnits: [] };
  }
};

export const getOrgWithChildren = async (id) => {
  try {
    const org = await db[TABLE_NAME].get({ id });
    if (!org) return null;

    const childrenOrgs = await db[TABLE_NAME]
      .where("path")
      .startsWith(org.path + "/")
      .toArray();

    org.children = childrenOrgs;
    return org;
  } catch (error) {
    console.error(`Failed to get organization unit with children:`, error);
    return null;
  }
};
```

### Step 7: Create OrganisationUnitLevelManager (`src/indexDB/OrganisationUnitLevelManager/index.js`)

```javascript
export const TABLE_FIELDS = "id, level, displayName";
export const TABLE_NAME = "organisationsUnitLevel";

export * from "./OrganisationUnitLevelManager";
```

### Step 8: Create OrganisationUnitLevelManager Implementation (`src/indexDB/OrganisationUnitLevelManager/OrganisationUnitLevelManager.js`)

```javascript
import db from "../db";
import { metadataApi } from "@/api";
import { TABLE_NAME } from ".";

export const pull = async () => {
  try {
    console.log("Downloading organization unit levels...");
    await db[TABLE_NAME].clear();
    
    const orgLevels = await metadataApi.getOrgUnitLevels();

    if (orgLevels && orgLevels.organisationUnitLevels.length > 0) {
      await addOrganisationUnitLevels(orgLevels.organisationUnitLevels);
      console.log("Organization unit levels stored successfully");
    }
  } catch (error) {
    console.error(`Failed to download organization unit levels:`, error);
    throw error;
  }
};

export const addOrganisationUnitLevels = async (orgLevels) => {
  try {
    await db[TABLE_NAME].bulkPut(orgLevels);
  } catch (error) {
    console.error(`Failed to store organization unit levels:`, error);
    throw error;
  }
};

export const getAllOrganisationUnitLevels = async () => {
  try {
    const orgLevels = await db[TABLE_NAME].toArray();
    return { organisationUnitLevels: orgLevels };
  } catch (error) {
    console.error(`Failed to get organization unit levels:`, error);
    return { organisationUnitLevels: [] };
  }
};
```

### Step 9: Create EnrollmentManager (`src/indexDB/EnrollmentManager/index.js`)

```javascript
export const TABLE_FIELDS = 
  "++id, enrollment, updatedAt, orgUnit, trackedEntityType, program, " +
  "enrollmentStatus, trackedEntity, enrolledAt, occurredAt, incidentDate, " +
  "isFollowUp, isDeleted, isOnline, attribute, value";
export const TABLE_NAME = "enrollment";

export * from "./EnrollmentManager";
```

### Step 10: Create EnrollmentManager Implementation (`src/indexDB/EnrollmentManager/EnrollmentManager.js`)

```javascript
import db from "../db";
import { TABLE_NAME } from ".";
import { dataApi } from "@/api";
import * as programManager from "@/indexDB/ProgramManager/ProgramManager";
import moment from "moment";
import { chunk } from "lodash";

export const pull = async ({ handleDispatchCurrentOfflineLoading, offlineSelectedOrgUnits }) => {
  try {
    console.log("Downloading enrollments...");
    
    if (offlineSelectedOrgUnits && offlineSelectedOrgUnits.length > 0) {
      await db[TABLE_NAME].clear();
    }

    const programs = await programManager.getPrograms();

    for (let j = 0; j < offlineSelectedOrgUnits.length; j++) {
      const org = offlineSelectedOrgUnits[j];
      
      for (let i = 0; i < programs.length; i++) {
        const program = programs[i];
        let totalPages = 0;

        try {
          for (let page = 1; ; page++) {
            if (totalPages && page > totalPages) break;

            const result = await dataApi.get("/api/tracker/enrollments", {
              paging: true,
              totalPages: true,
              pageSize: 2000,
              page,
            }, [
              `orgUnit=${org.id}`,
              `program=${program.id}`,
              `ouMode=DESCENDANTS`,
              `includeDeleted=true`,
              `fields=enrollment,updatedAt,trackedEntityType,trackedEntity,program,status,orgUnit,enrolledAt,incidentDate,followup`,
            ]);

            if (!result.instances || result.instances.length === 0 || page > result.pageCount) {
              break;
            }

            console.log(`ENROLLMENT = ${program.id} (page=${page}/${result.pageCount}, count=${result.instances.length})`);

            const resultEnrollments = {
              ...result,
              enrollments: result.instances,
            };

            await persist(await beforePersist([resultEnrollments], program.id));
            totalPages = result.pageCount;
          }
        } catch (error) {
          console.log("Enrollment:pull", error);
          continue;
        }
      }

      if (handleDispatchCurrentOfflineLoading) {
        handleDispatchCurrentOfflineLoading({
          id: "enr",
          percent: ((j + 1) / offlineSelectedOrgUnits.length) * 100,
        });
      }
    }
    
    console.log("Enrollments stored successfully");
  } catch (error) {
    console.log("Enrollment:pull", error);
    throw error;
  }
};

export const beforePersist = async (result, program, isOnline = 1) => {
  const objects = [];
  const ids = [];

  for (const [_, data] of result.entries()) {
    const enrollments = data.enrollments;
    if (!enrollments) continue;

    for (const en of enrollments) {
      const enrollment = {
        enrollment: en.enrollment,
        updatedAt: en.updatedAt || moment().format("YYYY-MM-DD"),
        program: program,
        orgUnit: en.orgUnit,
        enrollmentStatus: en.status,
        trackedEntityType: en.trackedEntityType,
        trackedEntity: en.trackedEntity,
        enrolledAt: en.enrolledAt,
        incidentDate: en.incidentDate,
        isOnline,
        isFollowUp: en.followup ? 1 : 0,
        isDeleted: en.deleted ? 1 : 0,
      };

      ids.push(enrollment.enrollment);

      if (en.attributes && en.attributes.length > 0) {
        for (const at of en.attributes) {
          const value = Object.assign({}, enrollment, {
            attribute: at.attribute,
            value: at.value,
          });
          objects.push(value);
        }
      } else {
        objects.push(enrollment);
      }
    }
  }

  // Delete existing records in chunks
  const partitions = chunk(ids, 200);
  for (const partition of partitions) {
    await db[TABLE_NAME].where("enrollment").anyOf(partition).delete();
  }

  return objects;
};

export const persist = async (enrollments) => {
  await db[TABLE_NAME].bulkPut(enrollments);
};

export const findOffline = async () => {
  return await db[TABLE_NAME].where("isOnline").anyOf(0).toArray();
};

export const setEnrollment = async ({ enrollment, program }) => {
  const enr = await beforePersist([{ enrollments: [enrollment] }], program, 0);
  await persist(enr);
};
```

## Phase 2: Database Initialization

### Step 11: Create Offline Initialization Component (`src/containers/ControlBar/PrepareOfflineModal.jsx`)

```javascript
import React, { useState } from "react";
import { Modal, Progress, Button, notification } from "antd";
import { useTranslation } from "react-i18next";

import * as meManager from "@/indexDB/MeManager/MeManager";
import * as organisationUnitLevelsManager from "@/indexDB/OrganisationUnitLevelManager/OrganisationUnitLevelManager";
import * as organisationUnitManager from "@/indexDB/OrganisationUnitManager/OrganisationUnitManager";
import * as programManager from "@/indexDB/ProgramManager/ProgramManager";
import * as trackedEntityManager from "@/indexDB/TrackedEntityManager/TrackedEntityManager";

const PrepareOfflineModal = ({ open, onCancel, onClose }) => {
  const { t } = useTranslation();
  const [loading, setLoading] = useState(false);
  const [ready, setReady] = useState(false);
  const [loadingProgress, setLoadingProgress] = useState({ id: null, percent: 0 });
  const [selectedOrgUnits, setSelectedOrgUnits] = useState({ selected: [] });

  const handleDispatchCurrentOfflineLoading = ({ id, percent }) => {
    setLoadingProgress({ id, percent });
  };

  const handleDownload = async () => {
    setLoading(true);
    const offlineSelectedOrgUnits = selectedOrgUnits.selected.map((path) => ({
      id: path.split("/").pop(),
    }));

    try {
      // Phase 1: Download metadata
      setLoadingProgress({ id: "metadata", percent: 0 });
      await meManager.pull();
      
      setLoadingProgress({ id: "metadata", percent: 15 });
      await organisationUnitLevelsManager.pull();
      
      setLoadingProgress({ id: "metadata", percent: 30 });
      await organisationUnitManager.pull();
      
      setLoadingProgress({ id: "metadata", percent: 70 });
      await programManager.pull(i18n.language);
      
      setLoadingProgress({ id: "metadata", percent: 100 });

      // Phase 2: Download data
      await trackedEntityManager.pullNested({ 
        handleDispatchCurrentOfflineLoading, 
        offlineSelectedOrgUnits 
      });

      setLoading(false);
      setReady(true);
      
      notification.success({
        message: t("offlineDataReady"),
        description: t("offlineDataDownloadedSuccessfully"),
      });
    } catch (error) {
      setLoading(false);
      notification.error({
        message: t("error"),
        description: t("failedToDownloadOfflineData"),
      });
    }
  };

  return (
    <Modal
      title={t("prepareOfflineMode")}
      open={open}
      onCancel={onCancel}
      footer={[
        <Button key="cancel" onClick={onCancel}>
          {t("cancel")}
        </Button>,
        <Button
          key="download"
          type="primary"
          loading={loading}
          onClick={handleDownload}
          disabled={selectedOrgUnits.selected.length === 0}
        >
          {t("downloadOfflineData")}
        </Button>,
        ready && (
          <Button key="ready" type="primary" onClick={onClose}>
            {t("ready")}
          </Button>
        ),
      ]}
    >
      {/* Organization Unit Selector Component */}
      <div>
        <h3>{t("selectOrganizationUnits")}</h3>
        {/* Add your organization unit selector component here */}
      </div>

      {/* Progress Display */}
      {loading && (
        <div>
          <h4>{t("downloadingData")}</h4>
          <Progress 
            percent={loadingProgress.percent} 
            status={loadingProgress.percent === 100 ? "success" : "active"}
          />
          <p>{t("currentOperation")}: {loadingProgress.id}</p>
        </div>
      )}

      {ready && (
        <div>
          <h4 style={{ color: "green" }}>{t("offlineDataReady")}</h4>
          <p>{t("youCanNowWorkOffline")}</p>
        </div>
      )}
    </Modal>
  );
};

export default PrepareOfflineModal;
```

## Phase 3: Usage and Testing

### Step 12: Create Database Utility Functions (`src/utils/offline.js`)

```javascript
import * as meManager from "@/indexDB/MeManager/MeManager";
import * as organisationUnitManager from "@/indexDB/OrganisationUnitManager/OrganisationUnitManager";
import * as organisationUnitLevelsManager from "@/indexDB/OrganisationUnitLevelManager/OrganisationUnitLevelManager";
import * as programManager from "@/indexDB/ProgramManager/ProgramManager";
import db from "@/indexDB/db";

export const getMetadataSet = (isOfflineMode) => {
  if (isOfflineMode) {
    return [
      programManager.getPrograms(),
      organisationUnitManager.getAllOrganisationUnits(),
      meManager.getMe(),
      organisationUnitLevelsManager.getAllOrganisationUnitLevels(),
      organisationUnitManager.getUserOrgs(),
    ];
  } else {
    // Return online API calls
    return [
      metadataApi.getPrograms(),
      metadataApi.get(`/api/organisationUnits`, {}, [
        "paging=false&fields=id,code,path,displayName,level,parent,translations&withinUserHierarchy=true",
      ]),
      metadataApi.getMe(),
      metadataApi.getOrgUnitLevels(),
      metadataApi.getUserOrgUnits(),
    ];
  }
};

export const findOffline = (TABLE_NAME) => 
  db[TABLE_NAME].where("isOnline").anyOf(0).toArray();

export const findChangedData = () =>
  Promise.all([
    findOffline("enrollment"), 
    findOffline("event"), 
    findOffline("trackedEntity")
  ]);

export const getOfflineDataStats = async () => {
  try {
    const stats = {
      user: await db.me.count(),
      organizationUnits: await db.organisationUnit.count(),
      organizationUnitLevels: await db.organisationsUnitLevel.count(),
      enrollments: await db.enrollment.count(),
      trackedEntities: await db.trackedEntity.count(),
      events: await db.event.count(),
      programs: await db.program.count(),
      optionSets: await db.optionSet.count(),
    };
    return stats;
  } catch (error) {
    console.error("Failed to get offline data statistics:", error);
    return {};
  }
};
```

### Step 13: Test the Implementation

```javascript
// Test script to verify database structure
import { db } from "@/indexDB/db";
import * as meManager from "@/indexDB/MeManager/MeManager";
import * as organisationUnitManager from "@/indexDB/OrganisationUnitManager/OrganisationUnitManager";

const testDatabase = async () => {
  try {
    console.log("Testing database structure...");
    
    // Test database connection
    await db.open();
    console.log("Database opened successfully");
    
    // Test user data
    const user = await meManager.getMe();
    console.log("User data:", user);
    
    // Test organization units
    const orgUnits = await organisationUnitManager.getUserOrgs();
    console.log("Organization units:", orgUnits);
    
    // Test database statistics
    const stats = await db.transaction('r', [db.me, db.organisationUnit], async () => {
      return {
        userCount: await db.me.count(),
        orgUnitCount: await db.organisationUnit.count(),
      };
    });
    
    console.log("Database statistics:", stats);
    console.log("Database structure test completed successfully!");
    
  } catch (error) {
    console.error("Database test failed:", error);
  }
};

// Run the test
testDatabase();
```

This step-by-step implementation provides a complete foundation for the IndexedDB database structure used in offline data collection. Each step builds upon the previous one, creating a robust offline-first architecture.
