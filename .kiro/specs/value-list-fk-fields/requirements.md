# Requirements Document

## Introduction

Georama currently has no concept of a "value list" (also called a codelist or foreign-key lookup field). A `Field` (`data_integration.models.Field`) whose underlying PostgreSQL column is a single-column foreign key to another table carries no metadata about that relationship anywhere in georama today — not in the model, not in the data integration sync, and not in what the OGC API Features endpoints serve.

This feature adds automatic detection of such foreign keys for Postgres-driver vector datasets, persists what is detected, exposes it in the schema and queryables georama serves, and adds a dedicated endpoint from which the id/name pairs of the referenced table can be fetched. The feature is backend-only: it makes the value list information available through georama's OGC API Features surface so that any client can render it as a lookup/dropdown; rendering it in gg-viewer's edit form is tracked separately and is explicitly out of scope here.

Detection runs only for `VectorDataSet` records whose `driver` is `POSTGRES`, because only those datasets carry a live PostgreSQL connection (`source_to_qsl.postgres`) that introspection can use. OGR-driver datasets have no such connection and are excluded. Detection is restricted to single-column foreign keys where the referenced table has a single-column primary key literally named `id` (case-insensitive) and a column literally named `name` (case-insensitive); composite foreign keys and tables without that shape are not detected as value lists.

The existing client-side contract (`OapifPropertySchema` in gg-viewer, already shipped and unused) declares an `x-orc-codelistUri` field alongside `enum`/`oneOf`. That field name is in fact the real `x-ogc-codelistUri` extension keyword defined by OGC API - Features - Part 5: Schemas (also published as OGC API - Common - Part 3, document 23-058r2) for referencing a codelist from a property's schema — it is not a project-invented contract. gg-viewer's current TypeScript type appears to have a one-letter typo (`orc` instead of `ogc`); correcting that client-side name is tracked separately and is out of scope here. Because a referenced table can hold far more rows than are reasonable to inline as a JSON Schema `enum`, this feature uses `x-ogc-codelistUri`: a detected value-list field is advertised with a URI pointing at a dedicated value-list data endpoint, not with inlined values. Consumption of that URI by any client, including gg-viewer, is out of scope for this specification.

## Glossary

- **Value_List**: the fact that a `Field`'s underlying PostgreSQL column is a single-column foreign key to a table that has a single-column primary key named `id` and a column named `name`, making that column suitable for rendering as a lookup of id/name pairs instead of free text.
- **Value_List_Field**: a `Field` for which a Value_List has been detected and persisted.
- **Foreign_Key (FK)**: a PostgreSQL `FOREIGN KEY` constraint on exactly one column of a table, referencing exactly one column of another (or the same) table.
- **Composite_Foreign_Key**: a `FOREIGN KEY` constraint spanning two or more columns. Explicitly out of scope for Value_List detection.
- **Ambiguous_Foreign_Key**: a `Field`'s underlying column that participates in a Composite_Foreign_Key, or in two or more separate single-column `FOREIGN KEY` constraints. FK_Detector reports no detection result for either case.
- **Referenced_Table**: the table a Foreign_Key points to, identified by PostgreSQL schema name and table name.
- **Id/Name_Convention**: the requirement that a Referenced_Table have a single-column primary key whose name is case-insensitively `id` and a column whose name is case-insensitively `name`. This is the sole shape FK_Detector recognizes in this feature.
- **FK_Detector**: the component that introspects PostgreSQL to find a Field's underlying column's Foreign_Key, if any, and checks the Referenced_Table against the Id/Name_Convention.
- **Value_List_Sync_Service**: the component, invoked from `DBService.integrate_project`, that runs FK_Detector for every Field of a `POSTGRES`-driver `VectorDataSet` and creates, updates, or deletes `FieldValueList` rows to match the current result.
- **FieldValueList**: the Django model, in `data_integration.models`, persisting a detected Value_List for one `Field`.
- **Field_Constraints_Builder**: the existing `getDatasetFieldConstraints` function (`features/views.py`) that builds the per-property schema dict (`field_constraints`) consumed by `GeoramaSqlProvider.get_fields` and, through it, by pygeoapi's queryables and schema endpoints.
- **Codelist**: the OGC API - Features - Part 5: Schemas (also OGC API - Common - Part 3, document 23-058r2) term for a referenced list of codes with definitions, expressed in a property's schema via the `x-ogc-codelistUri` keyword (profile `codelist-ref`) or inline via a `oneOf`/`const`/`title` structure (profile `codelist-inline`). This feature exposes Value_List_Field schemas using `x-ogc-codelistUri`, pointing at the Value_List_Endpoint, without populating `enum` (see Requirement 4).
- **Queryables**: the OGC API Features `/collections/{collectionId}/queryables` endpoint, served by `PygeoapiServer.collection_queryables`, describing which properties of a collection can be filtered on and their shape.
- **Schema endpoint**: the OGC API Features `/collections/{collectionId}/schema` endpoint, served by `PygeoapiServer.collection_schema`, describing the shape of a collection's properties.
- **Value_List_Endpoint**: the new endpoint this feature adds, serving the id/name rows of a Value_List_Field's Referenced_Table.
- **PygeoapiServer**: the existing Django view (`features/views.py`) dispatching all OGC API Features actions, including item create/update/delete.
- **GeoramaSqlProvider**: the existing pygeoapi provider (`features/pygeoapi_providers/sql.py`) wrapping `PostgreSQLProvider` for `POSTGRES`-driver datasets, whose `get_fields()` returns the `field_constraints` dict built by Field_Constraints_Builder.
- **PublishedAsOgcApiFeatures**: the existing model (`features.models`) representing one published OGC API Features collection, carrying permission checks (`has_read_permission`, `has_general_permission`) and an optional per-column permission mode (`column_permission`).
- **ColumnOgcApiFeatures**: the existing model (`features.models`) representing one published property of a `PublishedAsOgcApiFeatures` collection, one per `Field` of the underlying dataset, carrying its own permissions when `column_permission` is enabled.

## Requirements

### Requirement 1: Foreign key detection during project integration

**User Story:** As a georama operator, I want georama to automatically recognize foreign-key columns that follow the id/name convention, so that I do not have to configure value lists by hand for every lookup column in my QGIS projects.

#### Acceptance Criteria

1. WHEN `DBService.integrate_project` processes a `VectorDataSet` whose `driver.upper()` equals `POSTGRES`, THE Value_List_Sync_Service SHALL run FK_Detector for every `Field` of that dataset after that dataset's `Field` rows have been created or updated.
2. WHERE a `VectorDataSet`'s `driver.upper()` is not `POSTGRES`, THE Value_List_Sync_Service SHALL run no introspection for that dataset's `Field` rows.
3. THE FK_Detector SHALL open its introspection connection using the host, port, dbname, username, password, and schema recorded in that dataset's `source_to_qsl.postgres`, independent of Django's own default database connection.
4. THE FK_Detector SHALL issue, over the connection opened under Criterion 3, only read-only schema-introspection queries against PostgreSQL catalog information, and SHALL NOT execute any data-modifying statement over that connection.
5. THE FK_Detector SHALL treat a `Field`'s `name` as the name of a column of the table identified by the schema name and table name recorded in that dataset's `source_to_qsl.postgres`, consistent with how `Field` rows are derived from the live data source during QGIS project export.
6. IF a `Field`'s `name` does not match the name of any column currently present in the table identified under Criterion 5, for example because the table or column has been renamed, dropped, or recreated since that `Field` was last derived, THEN THE FK_Detector SHALL report no detection result for that `Field`, treating it as not having a Value_List.
7. WHEN a `Field`'s underlying column participates in exactly one single-column `FOREIGN KEY` constraint, THE FK_Detector SHALL identify the Referenced_Table of that constraint by PostgreSQL schema name and table name.
8. THE FK_Detector SHALL classify a Referenced_Table as satisfying the Id/Name_Convention only when that table has a primary key composed of exactly one column whose name, compared case-insensitively, is `id` — regardless of that column's PostgreSQL data type — and has at least one column, other than that primary key column, whose name, compared case-insensitively, is `name`.
9. WHEN a Referenced_Table satisfies the Id/Name_Convention, THE FK_Detector SHALL report the Referenced_Table's schema name, table name, the case-preserved name of its `id` column, and the case-preserved name of its `name` column, selected per Criterion 10 when applicable, as the detection result for that `Field`.
10. WHERE two or more columns of a Referenced_Table, other than its `id` column, each match the case-insensitive comparison to `name` described in Criterion 8, THE FK_Detector SHALL select, as the `name` column reported under Criterion 9, the one whose case-preserved column name sorts first in ascending Unicode code point order.
11. IF a `Field`'s underlying column participates in a `FOREIGN KEY` constraint spanning two or more columns (a Composite_Foreign_Key), or participates in two or more separate single-column `FOREIGN KEY` constraints (together with a Composite_Foreign_Key, an Ambiguous_Foreign_Key), THEN THE FK_Detector SHALL report no detection result for that `Field`, treating it as not having a Value_List.
12. IF a `Field`'s underlying column participates in no `FOREIGN KEY` constraint, or its Referenced_Table does not satisfy the Id/Name_Convention, THEN THE FK_Detector SHALL report no detection result for that `Field`.
13. WHERE the Referenced_Table of a `Field`'s Foreign_Key is the same table as the dataset's own underlying table (a self-referencing Foreign_Key), THE FK_Detector SHALL apply Criteria 8 and 9 identically to that case.
14. THE FK_Detector SHALL evaluate every `Field` of a dataset independently, so that two or more Fields of the same dataset referencing the same Referenced_Table, or referencing different Referenced_Tables, each receive their own, independently computed detection result.
15. IF FK_Detector's introspection connection does not succeed, or its introspection query does not complete, within 10 seconds for a dataset, THEN THE Value_List_Sync_Service SHALL abandon detection for that dataset's Fields for that run, SHALL emit a log record at WARNING level naming the dataset, and SHALL allow `DBService.integrate_project` to complete for the rest of the project.

### Requirement 2: Lifecycle of detected value lists across re-sync

**User Story:** As a georama operator, I want a previously detected value list to stay accurate as the underlying database schema evolves, so that stale or incorrect lookup metadata does not linger after I change a QGIS project or its database.

#### Acceptance Criteria

1. WHEN FK_Detector reports a detection result for a `Field` that has no existing FieldValueList row, THE Value_List_Sync_Service SHALL create one FieldValueList row for that `Field` holding that result.
2. WHEN FK_Detector reports a detection result for a `Field` that already has a FieldValueList row, and that result's referenced schema, referenced table, id column, or name column does not match byte-for-byte, case-sensitive, the corresponding value stored on that row, THE Value_List_Sync_Service SHALL update that existing row in place with the new result, SHALL create no second row for that `Field`.
3. WHEN FK_Detector reports no detection result for a `Field` that has an existing FieldValueList row, THE Value_List_Sync_Service SHALL delete that row.
4. THE Value_List_Sync_Service SHALL locate any existing FieldValueList row for a `Field` through that `Field`'s one-to-one relation before deciding whether to create, update, or delete, so repeated runs of `DBService.integrate_project` for an unchanged dataset leave the row count for that dataset's Fields unchanged.
5. IF FK_Detector's introspection for a dataset is abandoned under Requirement 1 Criterion 15, THEN THE Value_List_Sync_Service SHALL leave every existing FieldValueList row for that dataset's Fields unchanged for that run.
6. WHEN a `Field` is deleted, for example because `DBService.integrate_project`'s existing cleanup removes it after it is no longer present in the QGIS project config, THE database SHALL delete that `Field`'s FieldValueList row through the cascading delete of Requirement 3 Criterion 4, requiring no explicit cleanup code in Value_List_Sync_Service for that case.
7. WHEN Value_List_Sync_Service reconciles FieldValueList rows for a dataset during `DBService.integrate_project`, THE Value_List_Sync_Service SHALL perform every create, update, and delete operation for that dataset's FieldValueList rows inside the same database transaction as the `Field` synchronization that precedes it, SHALL leave every FieldValueList row for that dataset unchanged if that transaction is rolled back for any reason.
8. THE Value_List_Sync_Service SHALL require no cache invalidation, notification, or other synchronization step beyond the commit of that transaction for a create, update, or delete made during reconcile to become visible to any component that subsequently reads the affected FieldValueList row.

### Requirement 3: Value list storage model

**User Story:** As a developer, I want value-list detections persisted in a precisely defined model, so that the rest of georama can read a Field's value-list metadata without re-running introspection.

#### Acceptance Criteria

1. THE data_integration app SHALL define a `FieldValueList` model holding a one-to-one relation named `field` to `Field`, with `on_delete=CASCADE` and `related_name="value_list"`, a `Meta.verbose_name` and `Meta.verbose_name_plural` consistent with the other models declared in `data_integration.models`, and a `__str__` method that returns a human-readable identification of the row's `field` and `referenced_table`.
2. THE `FieldValueList` model SHALL hold `referenced_schema`, `referenced_table`, `id_column`, and `name_column` as character fields with `null=False` and `blank=False`, each with a maximum length of 63 characters matching PostgreSQL's identifier length limit, and each storing the case-preserved, non-empty name reported by FK_Detector.
3. THE `field` relation of `FieldValueList` SHALL be declared one-to-one, so the database enforces at most one `FieldValueList` row per `Field`, while enforcing no uniqueness constraint among `referenced_schema`, `referenced_table`, `id_column`, and `name_column` across different `FieldValueList` rows, so two or more `Field` rows may each hold a `FieldValueList` row referencing the same Referenced_Table.
4. WHEN a `Field` row is deleted, THE database SHALL delete its associated `FieldValueList` row automatically through the `on_delete=CASCADE` relation.
5. THE `FieldValueList` model SHALL hold a `last_synced` date-time field recording the moment that row was last created or updated.
6. WHEN Value_List_Sync_Service creates or updates a `FieldValueList` row, THE `FieldValueList` model SHALL set that row's `last_synced` field to the moment of that create or update.

### Requirement 4: Exposure of value-list metadata in served schema

**User Story:** As a client application consuming georama's OGC API Features endpoints, I want a value-list field's schema entry to carry a reference to where its lookup values can be fetched, so that I can render it as a lookup instead of free text without georama inlining a potentially large table of values.

#### Acceptance Criteria

1. WHEN Field_Constraints_Builder builds the schema entry for a property whose underlying `Field` has an associated `FieldValueList` row, THE Field_Constraints_Builder SHALL add to that property's schema dict a key literally named `x-ogc-codelistUri`, matching the keyword defined by OGC API - Features - Part 5: Schemas (OGC API - Common - Part 3, document 23-058r2) and declared by the `OapifPropertySchema` contract.
2. THE Field_Constraints_Builder SHALL set `x-ogc-codelistUri` to an absolute URL whose scheme and host are taken from `request.scheme` and `request.get_host()`, consistent with how `PygeoapiServer.handle_runtime_config` already builds `server_config["server"]["url"]`, and whose path is produced by resolving the `api-collection-valuelist` URL name declared in Requirement 5 Criterion 1 with `collection_id` set to the `identifier` of the publishing `PublishedAsOgcApiFeatures` and `property_name` set to that property's key in the schema dict, so that any character in `property_name` requiring percent-encoding is encoded exactly as Django's URL reversal encodes it.
3. IF a property's underlying `Field` has no associated `FieldValueList` row, THEN THE Field_Constraints_Builder SHALL leave `enum`, `oneOf`, and `x-ogc-codelistUri` absent from that property's schema dict, unchanged from current behaviour.
4. THE Field_Constraints_Builder SHALL leave `title`, `type`, `format`, `description`, `required`, `multipleOf`, `maximum`, and `maxLength` of a Value_List_Field's schema entry governed by the same rules as every other field, so the underlying column's declared OGC API type continues to describe the stored foreign-key value, not the Referenced_Table's row shape.
5. THE Georama API Features schema endpoint and queryables endpoint SHALL both serve the `x-ogc-codelistUri` key for a Value_List_Field, because both read their field descriptions from the same `field_constraints` dict built by Field_Constraints_Builder.
6. THE Field_Constraints_Builder SHALL never add an `x-ogc-codelistUri`, `enum`, or `oneOf` key to the geometry property's schema entry, because a geometry column cannot participate in a Foreign_Key and therefore never has an associated FieldValueList row.
7. THE Field_Constraints_Builder SHALL NOT populate `enum` on a Value_List_Field's schema entry, deliberately deviating from the `codelist-ref` profile's normal pairing of `enum` with `x-ogc-codelistUri`, because the Referenced_Table's cardinality is expected to be too large to enumerate safely in a schema or queryables response served on every request; `x-ogc-codelistUri` remains the sole means of discovering a Value_List_Field's valid values.

### Requirement 5: Value-list data endpoint

**User Story:** As a client application, I want an endpoint that returns the id/name pairs of a value-list field's referenced table, so that I can populate a lookup without needing direct database access.

#### Acceptance Criteria

1. THE features app SHALL serve HTTP GET at the path `/api/collections/<collection_id>/valuelist/<property_name>` relative to the features app root, named `api-collection-valuelist`.
2. WHEN a GET request arrives at that path for a `collection_id` matching a `PublishedAsOgcApiFeatures` and a `property_name` matching one of its `ColumnOgcApiFeatures` whose underlying `Field` has an associated `FieldValueList` row, THE Value_List_Endpoint SHALL query the Referenced_Table through that `PublishedAsOgcApiFeatures`'s dataset's own `source_to_qsl.postgres` connection, not through Django's own default database connection, and SHALL respond with HTTP status 200, a `Content-Type` response header of `application/json`, and a JSON object body holding a `codes` array whose entries are objects with a `code` key holding the Referenced_Table row's id column value and a `title` key holding that row's name column value, read from the Referenced_Table identified by that `FieldValueList` row, mirroring the `const`/`title` vocabulary the OGC standard's `codelist-inline` profile uses for codelist entries.
3. THE Value_List_Endpoint SHALL accept optional `limit` and `offset` query parameters, defaulting `limit` to 100 when absent and capping it at 1000, defaulting `offset` to 0 when absent, and SHALL return at most `limit` rows starting at `offset`.
4. IF `limit` or `offset` is supplied and is not a non-negative integer, or `limit` exceeds 1000, THEN THE Value_List_Endpoint SHALL respond with HTTP status 400 and an error body naming the invalid parameter, querying no Referenced_Table.
5. THE Value_List_Endpoint SHALL include in the JSON body a `numberMatched` field holding the total row count of the Referenced_Table, independent of `limit` and `offset`.
6. THE Value_List_Endpoint SHALL order the Referenced_Table rows it returns by the id column in ascending order, so repeated requests with the same `limit` and `offset` return the same page while the Referenced_Table's contents are unchanged.
7. WHEN more Referenced_Table rows exist beyond `offset + limit`, THE Value_List_Endpoint SHALL include in the JSON body a `links` array containing an entry whose `rel` is `next` and whose `href` is an absolute URL, built from the current request's scheme and host, pointing at this same endpoint's URL with `offset` advanced by `limit` and `limit` unchanged, mirroring the pagination convention already used by georama's OGC API Features item listing.
8. IF `collection_id` does not match any `PublishedAsOgcApiFeatures`, THEN THE Value_List_Endpoint SHALL respond with HTTP status 404, querying no Referenced_Table.
9. IF `property_name` does not match any `ColumnOgcApiFeatures` of the matched collection, or matches one whose underlying `Field` has no associated `FieldValueList` row, THEN THE Value_List_Endpoint SHALL respond with HTTP status 404, querying no Referenced_Table.
10. THE Value_List_Endpoint SHALL expose only the id and name column values of the Referenced_Table's rows in its response body, as the `code` and `title` of each entry in `codes`, exposing no other column of the Referenced_Table.
11. WHEN a GET request to the Value_List_Endpoint includes a query parameter other than `limit` or `offset`, THE Value_List_Endpoint SHALL ignore that parameter and SHALL process the request using only `limit` and `offset`, supporting no text-search or name-filter query parameter in this version.

### Requirement 6: Referential integrity on write

**User Story:** As a developer, I want write-path validation for value-list fields to rely on the database's own foreign-key constraint rather than duplicate it in application code, so that correctness does not depend on two independently maintained checks.

#### Acceptance Criteria

1. WHEN a create or update request through `PygeoapiServer` sets a Value_List_Field's value, THE GeoramaSqlProvider write path SHALL perform no existence check of that value against the Referenced_Table before submitting the write to PostgreSQL, relying on the Referenced_Table's PostgreSQL `FOREIGN KEY` constraint to enforce validity.
2. WHEN a create or update request sets a nullable Value_List_Field's value to NULL, THE GeoramaSqlProvider write path SHALL submit that write to PostgreSQL without the PostgreSQL `FOREIGN KEY` constraint rejecting it, consistent with standard SQL foreign-key semantics for a NULL referencing value.
3. IF a create or update request submits a non-NULL Value_List_Field value absent from its Referenced_Table, THEN PostgreSQL SHALL reject the write with an error whose SQLSTATE is `23503` (foreign_key_violation).
4. WHEN `PygeoapiServer`'s write path receives a database error whose SQLSTATE is `23503` for a create or update request, THE PygeoapiServer SHALL respond with HTTP status 409, and SHALL include in the response body the name of the property whose value was rejected, naming no column of the Referenced_Table and no constraint name in that response body.
5. THE PygeoapiServer SHALL respond with an HTTP status in the 5xx range for a database error whose SQLSTATE is not `23503`, SHALL NOT apply Criterion 4's HTTP 409 handling to any such error, so a not-null violation, a unique violation, or any other database error continues to surface as it does today.

### Requirement 7: Admin visibility of detected value lists

**User Story:** As a georama operator, I want to see which fields have a detected value list directly in the Django admin, so that I can confirm detection worked without querying the database myself.

#### Acceptance Criteria

1. WHEN an operator views a `VectorDataSet`'s read-only detail in the Django admin, and one of its `Field` rows has an associated `FieldValueList` row, THE VectorDataSetAdmin fields display SHALL show, within the existing `fields_detail` list entry for that `Field`, the referenced schema, referenced table, id column, name column, and `last_synced` timestamp held by that `FieldValueList` row, regardless of whether that `VectorDataSet` or that `Field` is published through `PublishedAsOgcApiFeatures`, `PublishedAsWfs`, both, or neither.
2. WHEN a `Field` row has no associated `FieldValueList` row, THE VectorDataSetAdmin fields display SHALL show no value-list information within that `Field`'s `fields_detail` list entry, unchanged from current behaviour, regardless of that `VectorDataSet`'s or `Field`'s publication status.
3. THE Django admin SHALL register no add, change, or delete form for `FieldValueList`, consistent with the existing read-only convention of `DataSetAdmin` and its subclasses.
4. THE VectorDataSetAdmin fields display SHALL HTML-escape the referenced schema, referenced table, id column, name column, and `last_synced` values before including them in the `fields_detail` HTML snippet, because those values originate from PostgreSQL database introspection rather than from static template text.

### Requirement 8: Permission model for the value-list data endpoint

**User Story:** As a security-conscious georama operator, I want viewing value-list data to require the same permission as viewing the field it belongs to, so that a value list cannot become an unauthenticated or under-authorized way to read a table that was never itself published.

#### Acceptance Criteria

1. WHEN a GET request arrives at the Value_List_Endpoint for a `collection_id` matching a `PublishedAsOgcApiFeatures`, THE Value_List_Endpoint SHALL evaluate `has_read_permission(request.user, central_app_label)` on that matched `PublishedAsOgcApiFeatures`, including when the request is unauthenticated, before resolving `property_name` to any `ColumnOgcApiFeatures` and before querying the Referenced_Table.
2. IF the evaluation of Criterion 1 returns false, THEN THE Value_List_Endpoint SHALL respond with HTTP status 403, resolving no `property_name` to any `ColumnOgcApiFeatures` and querying no Referenced_Table.
3. WHERE the matched `PublishedAsOgcApiFeatures` has `column_permission` set to true and `property_name` resolves to one of its `ColumnOgcApiFeatures`, THE Value_List_Endpoint SHALL evaluate `has_general_permission(request.user, central_app_label)` on that matched `ColumnOgcApiFeatures` before querying the Referenced_Table.
4. IF the evaluation of Criterion 3 returns false, THEN THE Value_List_Endpoint SHALL respond with HTTP status 403, querying no Referenced_Table.
5. THE Value_List_Endpoint SHALL grant access to a Value_List_Field's Referenced_Table rows based solely on the permissions evaluated in Criteria 1 and 3, consulting no permission defined on any separate publication of the Referenced_Table itself, so a Referenced_Table that is independently published as its own collection with different permissions does not alter what the Value_List_Endpoint serves.
6. THE Value_List_Endpoint SHALL leave the `User`, `PublishedAsOgcApiFeatures`, `ColumnOgcApiFeatures`, and `FieldValueList` tables unchanged for every GET request it serves, whether the permission evaluation of Criterion 1 or 3 succeeds or fails.

## Out of Scope

The following are explicitly excluded from this feature and recorded here so the exclusion is a decision rather than an omission:

- **Composite (multi-column) foreign keys.** Not detected; a `Field` whose column participates in one is treated as having no Value_List.
- **OGR-driver datasets.** FK_Detector never runs for them; they have no live PostgreSQL connection to introspect.
- **Tables without the Id/Name_Convention.** A Foreign_Key to a table lacking a single-column `id` primary key or a `name` column is not detected as a Value_List.
- **Configurable id/name column names.** This feature matches the literal names `id` and `name` (case-insensitively) only; making those names configurable per dataset or per field is a possible future extension, not built here.
- **A separate on-demand re-detection trigger.** Detection runs only as part of the existing `DBService.integrate_project` flow; no additional admin action is added to re-run it independently of a project sync.
- **Application-level pre-write existence checking of value-list values.** Covered by Requirement 6: georama relies on PostgreSQL's foreign-key constraint and surfaces its rejection, rather than querying the Referenced_Table before every write.
- **Any gg-viewer or other client rendering change.** This specification covers only what georama detects, stores, and serves; consuming `x-ogc-codelistUri` to render a dropdown is separate, already-tracked work.
- **Editing or overriding a detected value list through the Django admin or any other UI.** Admin exposure is read-only, consistent with the existing `DataSetAdmin` convention.
- **Manually configured fixed enumerations not backed by a real foreign key.** This feature detects only real PostgreSQL `FOREIGN KEY` constraints; a hand-authored fixed list of choices is a different feature.
- **A separate permission specifically for value-list data.** Requirement 8 ties access to the existing permission of the referencing field/layer rather than introducing a new permission type.
- **Correcting gg-viewer's `OapifPropertySchema.x-orc-codelistUri` field name.** That field name appears to be a typo of the real OGC keyword `x-ogc-codelistUri`. Fixing it is a client-side TypeScript change in the gg-viewer repository and is tracked separately from this backend-only specification.
