# Delta Sync Strategy for QSO Downloads

## Overview

Wavelog has **built-in support for delta synchronization** via the `last_modified` timestamp column. This enables efficient offline-sync workflows where only changed records are transferred.

## Existing Infrastructure

### 1. Database Tracking

All QSO records automatically track modification timestamps:
- **Column**: `last_modified` (TIMESTAMP)
- **Behavior**: Auto-updates on every INSERT or UPDATE
- **Implementation**: Migration `256_crea_modidates.php` added this to all tables including the QSO table

### 2. Existing Export Methods

#### `get_contacts_adif` Endpoint (Already Implemented)
- **Path**: `POST /index.php/api/get_contacts_adif`
- **Purpose**: Export QSOs incrementally by QSO ID
- **Parameters**:
  - `key`: API key (required)
  - `station_id`: Station to export (required)
  - `fetchfromid`: QSO ID threshold (returns QSOs with ID > this value)
  - `limit`: Max QSOs to fetch (default 20000)

**Current Limitation**: Uses `COL_PRIMARY_KEY > fetchfromid` (ID-based), not timestamp-based.

#### `Adif_data` Model Methods
Located in `application/models/Adif_data.php`:

- **`export_all_chunked()`** - All QSOs with optional date range filtering
- **`export_custom_chunked()`** - QSOs with from/to date filtering
- **`export_past_id_chunked()`** - QSOs with ID > threshold (used by `/api/get_contacts_adif`)

## Recommended Delta Sync Approach

### Option 1: Use `last_modified` Timestamp (Recommended for Offline Apps)

**Advantage**: Only downloads records that actually changed, not dependent on sequential IDs.

Create a new API endpoint `/api/qso_sync_delta`:

```json
POST /index.php/api/qso_sync_delta
{
  "key": "API_KEY",
  "station_id": 1,
  "since": "2026-02-01T10:30:00",  // ISO 8601 timestamp
  "limit": 1000
}
```

**Response**:
```json
{
  "status": "success",
  "qsos": [...ADIF records...],
  "last_sync_timestamp": "2026-02-01T15:45:30",
  "total_changed": 150,
  "has_more": false
}
```

**Implementation**: Query where `last_modified >= since_timestamp` and `station_id = ?`

### Option 2: Use Existing `/api/get_contacts_adif` (Already Available)

**Current Workflow**:
1. Store `lastfetchedid` locally after each sync
2. Call `/api/get_contacts_adif` with `fetchfromid: lastfetchedid`
3. API returns all QSOs with ID > lastfetchedid
4. Store returned `lastfetchedid` for next sync

**Limitation**: Requires knowing the last QSO ID, doesn't distinguish between "new" and "changed" records.

**Benefit**: Already implemented and working.

## Hybrid Approach (Best for Your Use Case)

### Mobile App Offline Sync Workflow

1. **Initial Full Sync** (first time only):
   ```bash
   curl -X POST "https://yourserver.com/index.php/api/get_contacts_adif" \
     -H "Content-Type: application/json" \
     -d '{
       "key": "YOUR_KEY",
       "station_id": 1,
       "fetchfromid": 0,
       "limit": 50000
     }'
   ```
   - Store all QSOs locally
   - Store last received `lastfetchedid`
   - Store current server timestamp as `last_sync_time`

2. **Incremental Sync** (every app launch):
   ```bash
   # Option A: Use ID-based delta (simpler, works with current API)
   curl -X POST "https://yourserver.com/index.php/api/get_contacts_adif" \
     -H "Content-Type: application/json" \
     -d '{
       "key": "YOUR_KEY",
       "station_id": 1,
       "fetchfromid": 1234,  # Last QSO ID from previous sync
       "limit": 5000
     }'
   ```

3. **Upload Local Changes**:
   ```bash
   curl -X POST "https://yourserver.com/index.php/api/qso_update" \
     -H "Content-Type: application/json" \
     -d '{
       "key": "YOUR_KEY",
       "id": 1234,
       "comment": "Updated locally",
       "rst_sent": "599"
     }'
   ```

## Query Examples (for reference)

### Direct Database: Find All Changed QSOs Since Timestamp

```sql
SELECT COL_PRIMARY_KEY, COL_CALL, last_modified 
FROM qso_table 
WHERE station_id = 1 
  AND last_modified >= '2026-02-01 10:30:00'
ORDER BY last_modified ASC
LIMIT 1000;
```

### CodeIgniter Query (for a new API method):

```php
function get_modified_qsos($station_id, $since_timestamp, $limit = 1000) {
    $this->db->select('*');
    $this->db->from($this->config->item('table_name'));
    $this->db->where('station_id', $station_id);
    $this->db->where('last_modified >=', $since_timestamp);
    $this->db->order_by('last_modified', 'ASC');
    $this->db->limit($limit);
    
    return $this->db->get();
}
```

## Summary Table: Which Approach for Your App?

| Method | Use Case | Already Implemented? |
|--------|----------|---------------------|
| ID-based delta (`fetchfromid`) | New QSOs only, simple workflow | ✅ Yes (`get_contacts_adif`) |
| Timestamp-based delta (`last_modified`) | Changed QSOs (edits + new), accurate sync | ❌ No (but infrastructure exists) |
| Full re-sync | Debugging, reconciliation | ✅ Yes (`get_contacts_adif` with `fetchfromid=0`) |

## Recommendation for Your Smartphone App

**Use the existing `/api/get_contacts_adif` endpoint** with ID-based delta:
- ✅ Already implemented and tested
- ✅ Works with `/api/qso_update` we just built
- ✅ Simple local tracking (just store last QSO ID)
- ✅ Efficient for typical mobile workflows

**If you later need true change-tracking** (e.g., server-side edits by other operators):
- Implement new endpoint using `last_modified` column
- Falls back on existing database infrastructure (column already exists)

## Files Reference

- **Database column**: Added by migration [256_crea_modidates.php](../application/migrations/256_crea_modidates.php)
- **Export methods**: [application/models/Adif_data.php](../application/models/Adif_data.php)
- **QSO class**: [src/QSLManager/QSO.php](../src/QSLManager/QSO.php) (tracks `$last_modified`)
- **API endpoint**: [application/controllers/Api.php](../application/controllers/Api.php) (`get_contacts_adif` method)
