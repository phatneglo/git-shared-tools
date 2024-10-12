# README: Userpriv Class Modifications

This document outlines the modifications made to the `Userpriv` class in the `PHPMaker2024\UserAccessControl` namespace. The changes focus on customizing the system with user levels, particularly in the `run()` method, `editRow()` method, and privilege handling.

## Step-by-Step Modifications

### 1. Loading User Level Settings (Lines 200-211)

The code now loads user level settings based on the selected user level ID:

```php
if ($user_level_id = Get("user_level_id")) {
    $systemInfo = ExecuteRow("SELECT
            systems.level_permissions, 
            user_levels.user_level_id
    FROM
            systems
            INNER JOIN
            user_levels
            ON 
                    systems.system_id = user_levels.system_id
    WHERE
            user_levels.user_level_id = $user_level_id");
    if (!empty($systemInfo["level_permissions"])) {
            $ar = json_decode($systemInfo["level_permissions"], true);
    }
}
```

This modification allows the system to load level permissions specific to the selected user level.

### 2. Handling Post Action (Lines 256-306)

The `run()` method has been updated to handle the "update" action differently:

```php
} else {
    if (Post("action") == "update") {
        $this->CurrentAction = Post("action");
        
        // Get the user_level_id from the form
        $user_level_id = Post("x_user_level_id");
        $this->user_level_id->setFormValue($user_level_id);
        
        // Load the level_permissions for this user_level_id
        $systemInfo = ExecuteRow("SELECT
                systems.level_permissions,
                user_levels.user_level_id
        FROM
                systems
                INNER JOIN
                user_levels
                ON
                        systems.system_id = user_levels.system_id
        WHERE
                user_levels.user_level_id = " . $user_level_id);
        
        if (!empty($systemInfo["level_permissions"])) {
            $permissionsArray = json_decode($systemInfo["level_permissions"], true);
        } else {
            $permissionsArray = [];
        }
        
        // Process the privileges
        $this->Privileges = [];
        
        foreach ($permissionsArray as $tableInfo) {
            $tableName = $tableInfo[0];
            $listName = $tableInfo[5] ?? $tableInfo[1];
            
            $privilege = 0;
            $postIndex = array_search($tableName, array_column($permissionsArray, 0));
            
            if ($postIndex !== false) {
                $privilege += (int)Post("add_" . $postIndex, 0);
                $privilege += (int)Post("delete_" . $postIndex, 0);
                $privilege += (int)Post("edit_" . $postIndex, 0);
                $privilege += (int)Post("list_" . $postIndex, 0);
                $privilege += (int)Post("view_" . $postIndex, 0);
                $privilege += (int)Post("search_" . $postIndex, 0);
                $privilege += (int)Post("admin_" . $postIndex, 0);
                $privilege += (int)Post("import_" . $postIndex, 0);
                $privilege += (int)Post("lookup_" . $postIndex, 0);
                $privilege += (int)Post("export_" . $postIndex, 0);
                $privilege += (int)Post("push_" . $postIndex, 0);
            }
            
            $this->Privileges[$listName] = $privilege;
        }
    }
}

```

This change allows for more dynamic handling of user level permissions based on the system's configuration.

### 3. Updated editRow() Method (Lines 426-486)

The `editRow()` method has been significantly modified to work with the new permission structure:

```php
protected function editRow()
{
    global $Security;
    $c = Conn(Config("USER_LEVEL_PRIV_DBID"));

    // Fetch level_permissions for the selected user_level_id
    $sql = "SELECT s.level_permissions 
            FROM systems s
            INNER JOIN user_levels ul ON s.system_id = ul.system_id
            WHERE ul.user_level_id = " . $this->user_level_id->CurrentValue;
    
    $levelPermissions = ExecuteScalar($sql);
    
    if ($levelPermissions === false) {
        // Handle error - couldn't fetch level_permissions
        return false;
    }

    $permissionsArray = json_decode($levelPermissions, true);
    if (!is_array($permissionsArray)) {
        // Handle error - invalid JSON in level_permissions
        return false;
    }

    $success = true;

    foreach ($permissionsArray as $tableInfo) {
        $tableName = $tableInfo[4] . $tableInfo[0]; // Construct table name from prefix and name
        $listName = $tableInfo[5] ?? $tableInfo[1]; // Get the list name, fallback to the second element if not set
        
        // Get privilege from $this->Privileges, default to 0 if not set
        $privilege = $this->Privileges[$listName] ?? 0;

        // Update or insert privilege
        $sql = "UPDATE " . Config("USER_LEVEL_PRIV_TABLE") . " ... ";
        $result = Execute($sql);

        if ($result === false) {
            $success = false;
            break;
        }

        // If no rows were updated, insert a new record
        if ($result == 0) {
            $sql = "INSERT INTO " . Config("USER_LEVEL_PRIV_TABLE") . " ... ";
            $result = Execute($sql);
            if ($result === false) {
                $success = false;
                break;
            }
        }
    }

    if ($success) {
        $Security->setupUserLevel();
        return true;
    } else {
        return false;
    }
}
```

This updated method now works with the new permission structure, updating or inserting privileges based on the loaded level permissions.

## Key Changes

1. The system now loads user level permissions from the `systems` table, allowing for more flexible permission management.
2. The privilege processing in the `run()` method has been updated to work with the new permission structure.
3. The `editRow()` method now updates or inserts privileges based on the loaded level permissions, rather than using a fixed set of table permissions.

These modifications allow for a more dynamic and customizable user level system, where permissions can be managed on a per-system basis rather than being hardcoded for specific tables.
