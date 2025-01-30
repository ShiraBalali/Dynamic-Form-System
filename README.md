# Dynamic-Form-System

## Original Schema
![image](https://github.com/user-attachments/assets/6caa0623-1938-42f8-b448-466fa4fa9348)


### Tables:
- Users
  - UserID (int, PK)
  - UserName (nvarchar(30))
  - FirstName (nvarchar(30))
  - LastName (nvarchar(30))

- Forms
  - FormID (int, PK)
  - CreatorUserID (int, FK to Users)
  - Name (nvarchar(50))

- Fields
  - FieldID (int, PK)
  - FormID (int, FK to Forms)
  - Name (nvarchar(100))
  - Type (varchar(10))

- FieldOptions
  - OptionID (int, PK)
  - FieldID (int, FK to Fields)
  - OptionText (nvarchar(100))

- Fill
  - FillID (int, PK)
  - FormID (int, FK to Forms)
  - CreatorUserID (int, FK to Users)
  - CreateDate (datetime)
  - LastUpdateDate (datetime)

- FillNumContent
  - ContentID (int, PK)
  - FillID (int, FK to Fill)
  - FieldID (int, FK to Fields)
  - NumValue (numeric(38,10))

- FillTextContent
  - ContentID (int, PK)
  - FillID (int, FK to Fill)
  - FieldID (int, FK to Fields)
  - TextValue (nvarchar(max))

- FillMultiContent
  - ContentID (int, PK)
  - FillID (int, FK to Fill)
  - FieldID (int, FK to Fields)
  - OptionID (int, FK to FieldOptions)
 
## New Schema Changes:

- FillNumContent
  - ContentID (int, PK)
  - FillID (int, FK to Fill)
  - FieldID (int, FK to Fields)
  - NumValue (numeric(38,10))
  - Version (int)

- FillTextContent
  - ContentID (int, PK)
  - FillID (int, FK to Fill)
  - FieldID (int, FK to Fields)
  - TextValue (nvarchar(max))
  - Version (int)

- FillMultiContent
  - ContentID (int, PK)
  - FillID (int, FK to Fill)
  - FieldID (int, FK to Fields)
  - OptionID (int, FK to FieldOptions)
  - Version (int)
 

### What we did:

Added a "Version" number column to each content table (FillNumContent, FillTextContent, FillMultiContent).
Each field value in a form can now track its changes over time.

### How it works:

#### When someone first fills out a form:

- Create rows with Version=1 for each field they filled.

#### When someone changes a field:

- Keep the old row as is.
- Create a new row for just that field with Version++.
- Update the Fill.LastUpdateDate.

### Example
```
If a form has 2 fields: Name and Age
First submission (Version 1):
- Name: "Milo" 
- Age: 4

User changes Age to 5:
- Name: "Milo" (Version 1) - unchanged, no new row needed
- Age: 4 (Version 1) - kept for history
- Age: 5 (Version 2) - new row added
 ```

### Getting a specific version:
#### To see how the form looked at Version 2:

For each field, find the row with the highest version number that's not bigger than 2
In our example:

- Name: Version 1 
- Age: Version 2

# API Requests

## Get Fill History
Retrieves a single fill document with its complete version history.
```
GET /api/fills/123/history
```

```
200 OK
{
  "fillId": 123,
  "formId": 456,
  "formName": "Dog Introduction",
  "createDate": "25/1/2025 8:00",
  "lastUpdateDate": "29/1/2025 12:00",
  "currentVersion": 3,
  "fields": {
    "name": {
      "fieldId": 1,
      "type": "text",
      "versions": [
        {
          "version": 1,
          "value": "Milo"
        },
        {
          "version": 2,
          "value": "Jungo"
        }
      ]
    },
    "age": {
      "fieldId": 2,
      "type": "number",
      "versions": [
        {
          "version": 1,
          "value": 4
        },
        {
          "version": 3,
          "value": 5
        }
      ]
    },
    "toys": {
      "fieldId": 3,
      "type": "multi",
      "versions": [
        {
          "version": 1,
          "values": ["bone", "teddy"]
        },
        {
          "version": 3,
          "values": ["bone", "toro", "teddy"]
        }
      ]
    }
  }
}
```


## Get Version Comparison
Retrieves a specific version with highlighted changes compared to previous version.
```
GET /api/fills/123/version/3
```

```
200 OK
{
  "fillId": 123,
  "version": 3,
  "fields": {
    "name": {
      "fieldId": 1,
      "type": "text",
      "value": "Jungo",
      "changed": false
    },
    "age": {
      "fieldId": 2,
      "type": "number",
      "value": 5,
      "changed": true
    },
    "toys": {
      "fieldId": 3,
      "type": "multi",
      "value": ["bone", "toro", "teddy"],
      "changed": true
    }
  }
}
```

## Errors
Retrieves a single fill document with invalid fill id.
```
GET /api/fills/154/history
```

```
404 Not Found
{
  "error": "NOT_FOUND",
  "message": "Fill ID 154 does not exist"
}
```

Retrieves a single fill document with invalid version.
```
GET /api/fills/123/version/8
```

```
404 Not Found
{
  "error": "NOT_FOUND",
  "message": "Version 8 does not exist for Fill ID 123"
}
```

Deletes a form with a regular user (not an admin)
```
DELETE /api/form/456
```

```
403 Forbidden
{
  "error": "FORBIDDEN",
  "message": "You don't have permission to delete this form"
}
```


# Alternative solution:
## New Schema Changes:

 - Snapshots
    - SnapshotID (PK)
    - FillID (FK to Fill)
    - Version (int)
    - CreateDate (datetime)

 - FillEntry
    - EntryID (PK)
    - SnapshotID (FK to Snapshots)
    - FieldID (FK to Fields)
    - ContentReference (FK to FillNumContent/FillTextContent/FillMultiContent)

## New Logic
Instead of adding versions to content tables, we use two new tables:
- Snapshots - marks each version of a form fill.
- FillEntry - maps each field in a snapshot to its content.

When a form is updated:
- Create new content records only for changed fields.
- Create a new snapshot.
- Create FillEntries pointing to new content for changed fields and reusing existing content references for unchanged fields.

## Final Thoughts
Adding a Version column and the Snapshot-Entry approach both give us perfect snapshots of the form.

 The difference is where the complexity lives - the Version approach needs more work in the backend API to piece together the form state at any version, while the Snapshot-Entry approach makes querying easier but has more complex database structure.

 I chose the Version approach since it's simpler to understand and maintain, even though it needs more API logic.
