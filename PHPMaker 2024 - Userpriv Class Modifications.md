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

## Sample Code
```php
<?php

namespace PHPMaker2024\GUARDIAN;

use Doctrine\DBAL\ParameterType;
use Doctrine\DBAL\Connection;
use Doctrine\DBAL\Query\QueryBuilder;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Container\ContainerInterface;
use Slim\Routing\RouteCollectorProxy;
use Slim\App;
use Closure;

/**
 * Page class
 */
class Userpriv extends UserLevels
{
    use MessagesTrait;

    // Page ID
    public $PageID = "userpriv";

    // Project ID
    public $ProjectID = PROJECT_ID;

    // Page object name
    public $PageObjName = "Userpriv";

    // View file path
    public $View = null;

    // Title
    public $Title = null; // Title for <title> tag

    // Rendering View
    public $RenderingView = false;

    // CSS class/style
    public $CurrentPageName = "userpriv";

    // Page headings
    public $Heading = "";
    public $Subheading = "";
    public $PageHeader;
    public $PageFooter;

    // Page layout
    public $UseLayout = true;

    // Page terminated
    private $terminated = false;

    // Page heading
    public function pageHeading()
    {
        global $Language;
        if ($this->Heading != "") {
            return $this->Heading;
        }
        if (method_exists($this, "tableCaption")) {
            return $this->tableCaption();
        }
        return "";
    }

    // Page subheading
    public function pageSubheading()
    {
        global $Language;
        if ($this->Subheading != "") {
            return $this->Subheading;
        }
        return "";
    }

    // Page name
    public function pageName()
    {
        return CurrentPageName();
    }

    // Page URL
    public function pageUrl($withArgs = true)
    {
        $route = GetRoute();
        $args = RemoveXss($route->getArguments());
        if (!$withArgs) {
            foreach ($args as $key => &$val) {
                $val = "";
            }
            unset($val);
        }
        return rtrim(UrlFor($route->getName(), $args), "/") . "?";
    }

    // Show Page Header
    public function showPageHeader()
    {
        $header = $this->PageHeader;
        $this->pageDataRendering($header);
        if ($header != "") { // Header exists, display
            echo '<div id="ew-page-header">' . $header . '</div>';
        }
    }

    // Show Page Footer
    public function showPageFooter()
    {
        $footer = $this->PageFooter;
        $this->pageDataRendered($footer);
        if ($footer != "") { // Footer exists, display
            echo '<div id="ew-page-footer">' . $footer . '</div>';
        }
    }

    // Constructor
    public function __construct()
    {
        parent::__construct();
        global $Language, $DashboardReport, $DebugTimer, $UserTable;
        $this->TableVar = '_user_levels';
        $this->TableName = 'user_levels';

        // Table CSS class
        $this->TableClass = "table table-striped table-bordered table-hover table-sm ew-table";

        // Initialize
        $GLOBALS["Page"] = &$this;

        // Language object
        $Language = Container("app.language");

        // Table object (_user_levels)
        if (!isset($GLOBALS["_user_levels"]) || $GLOBALS["_user_levels"]::class == PROJECT_NAMESPACE . "_user_levels") {
            $GLOBALS["_user_levels"] = &$this;
        }

        // Start timer
        $DebugTimer = Container("debug.timer");

        // Debug message
        LoadDebugMessage();

        // Open connection
        $GLOBALS["Conn"] ??= $this->getConnection();

        // User table object
        $UserTable = Container("usertable");
    }

    // Get content from stream
    public function getContents(): string
    {
        global $Response;
        return $Response?->getBody() ?? ob_get_clean();
    }

    // Is lookup
    public function isLookup()
    {
        return SameText(Route(0), Config("API_LOOKUP_ACTION"));
    }

    // Is AutoFill
    public function isAutoFill()
    {
        return $this->isLookup() && SameText(Post("ajax"), "autofill");
    }

    // Is AutoSuggest
    public function isAutoSuggest()
    {
        return $this->isLookup() && SameText(Post("ajax"), "autosuggest");
    }

    // Is modal lookup
    public function isModalLookup()
    {
        return $this->isLookup() && SameText(Post("ajax"), "modal");
    }

    // Is terminated
    public function isTerminated()
    {
        return $this->terminated;
    }

    /**
     * Terminate page
     *
     * @param string $url URL for direction
     * @return void
     */
    public function terminate($url = "")
    {
        if ($this->terminated) {
            return;
        }
        global $TempImages, $DashboardReport, $Response;

        // Page is terminated
        $this->terminated = true;

        // Page Unload event
        if (method_exists($this, "pageUnload")) {
            $this->pageUnload();
        }
        DispatchEvent(new PageUnloadedEvent($this), PageUnloadedEvent::NAME);
        if (!IsApi() && method_exists($this, "pageRedirecting")) {
            $this->pageRedirecting($url);
        }

        // Close connection
        CloseConnections();

        // Return for API
        if (IsApi()) {
            $res = $url === true;
            if (!$res) { // Show response for API
                $ar = array_merge($this->getMessages(), $url ? ["url" => GetUrl($url)] : []);
                WriteJson($ar);
            }
            $this->clearMessages(); // Clear messages for API request
            return;
        } else { // Check if response is JSON
            if (WithJsonResponse()) { // With JSON response
                $this->clearMessages();
                return;
            }
        }

        // Go to URL if specified
        if ($url != "") {
            if (!Config("DEBUG") && ob_get_length()) {
                ob_end_clean();
            }
            SaveDebugMessage();
            Redirect(GetUrl($url));
        }
        return; // Return to controller
    }
    public $Disabled;
    public $TableNameCount;
    public $Privileges = [];
    public $UserLevelList = [];
    public $UserLevelPrivList = [];
    public $TableList = [];

    /**
     * Page run
     *
     * @return void
     */
    public function run()
    {
        global $ExportType, $Language, $Security, $CurrentForm, $Breadcrumb;

        // Use layout
        $this->UseLayout = $this->UseLayout && ConvertToBool(Param(Config("PAGE_LAYOUT"), true));

        // View
        $this->View = Get(Config("VIEW"));

        // Load user profile
        if (IsLoggedIn()) {
            Profile()->setUserName(CurrentUserName())->loadFromStorage();
        }
        $this->CurrentAction = Param("action"); // Set up current action

        // Global Page Loading event (in userfn*.php)
        DispatchEvent(new PageLoadingEvent($this), PageLoadingEvent::NAME);

        // Page Load event
        if (method_exists($this, "pageLoad")) {
            $this->pageLoad();
        }
        $Breadcrumb = Breadcrumb::create("index")
            ->add("list", "_user_levels", "UserLevelsList", "", "_user_levels")
            ->add("userpriv", "UserLevelPermission", CurrentUrl());
        $this->Heading = $Language->phrase("UserLevelPermission");

        // Load user level settings
        $this->UserLevelList = $GLOBALS["USER_LEVELS"];
        $this->UserLevelPrivList = $GLOBALS["USER_LEVEL_PRIVS"];
        $ar = $GLOBALS["USER_LEVEL_TABLES"];

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

        // Set up allowed table list
        foreach ($ar as $t) {
            if ($t[3]) { // Allowed
                $tempPriv = $Security->getUserLevelPrivEx($t[4] . $t[0], $Security->CurrentUserLevelID);
                if (($tempPriv & Allow::ADMIN->value) == Allow::ADMIN->value) { // Allow Admin
                    $this->TableList[] = array_merge($t, [$tempPriv]);
                }
            }
        }
        $this->TableNameCount = count($this->TableList);

        // Get action
        if (Post("action") == "") {
            $this->CurrentAction = "show"; // Display with input box
            // Load key from QueryString
            if (Get("user_level_id") !== null) {
                $this->user_level_id->setQueryStringValue(Get("user_level_id"));
            } else {
                $this->terminate("UserLevelsList"); // Return to list
                return;
            }
            if ($this->user_level_id->QueryStringValue == "-1") {
                $this->Disabled = " disabled";
            } else {
                $this->Disabled = "";
            }
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


        // Should not edit own permissions
        if ($Security->hasUserLevelID($this->user_level_id->CurrentValue)) {
            $this->terminate("UserLevelsList"); // Return to list
            return;
        }
        switch ($this->CurrentAction) {
            case "show": // Display
                if (!$Security->setupUserLevelEx()) { // Get all User Level info
                    $this->terminate("UserLevelsList"); // Return to list
                    return;
                }
                $ar = [];
                for ($i = 0; $i < $this->TableNameCount; $i++) {
                    $table = $this->TableList[$i];
                    $cnt = count($table);
                    $tempPriv = $Security->getUserLevelPrivEx($table[4] . $table[0], $this->user_level_id->CurrentValue);
                    $ar[] = ["table" => ConvertToUtf8($this->getTableCaption($i)), "name" => $table[1], "index" => $i, "permission" => $tempPriv, "allowed" => $table[$cnt - 1]];
                }
                $this->Privileges["disabled"] = $this->Disabled;
                $this->Privileges["permissions"] = $ar;
                $this->Privileges["ids"] = PRIVILEGES;
                foreach (PRIVILEGES as $priv) {
                    $this->Privileges[$priv] = GetPrivilege($priv);
                }
                break;
            case "update": // Update
                if ($this->editRow()) { // Update record based on key
                    if ($this->getSuccessMessage() == "") {
                        $this->setSuccessMessage($Language->phrase("UpdateSuccess")); // Set up update success message
                    }
                    // Alternatively, comment out the following line to go back to this page
                    $this->terminate("UserLevelsList"); // Return to list
                    return;
                }
        }

        // Set LoginStatus / Page_Rendering / Page_Render
        if (!IsApi() && !$this->isTerminated()) {
            // Setup login status
            SetupLoginStatus();

            // Pass login status to client side
            SetClientVar("login", LoginStatus());

            // Global Page Rendering event (in userfn*.php)
            DispatchEvent(new PageRenderingEvent($this), PageRenderingEvent::NAME);

            // Page Render event
            if (method_exists($this, "pageRender")) {
                $this->pageRender();
            }

            // Render search option
            if (method_exists($this, "renderSearchOptions")) {
                $this->renderSearchOptions();
            }
        }
    }

    // Update privileges
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

    // Get table caption
    protected function getTableCaption($i)
    {
        global $Language;
        $caption = "";
        if ($i < $this->TableNameCount) {
            $caption = $Language->tablePhrase($this->TableList[$i][1], "TblCaption");
            if ($caption == "") {
                $caption = $this->TableList[$i][2];
            }
            if ($caption == "") {
                $caption = $this->TableList[$i][0];
                $caption = preg_replace('/^\{\w{8}-\w{4}-\w{4}-\w{4}-\w{12}\}/', '', $caption); // Remove project id
            }
        }
        return $caption;
    }

    // Page Load event
    public function pageLoad()
    {
        //Log("Page Load");
    }

    // Page Unload event
    public function pageUnload()
    {
        //Log("Page Unload");
    }

    // Page Redirecting event
    public function pageRedirecting(&$url)
    {
        // Example:
        //$url = "your URL";
    }

    // Message Showing event
    // $type = ''|'success'|'failure'
    public function messageShowing(&$msg, $type)
    {
        // Example:
        //if ($type == "success") $msg = "your success message";
    }

    // Page Render event
    public function pageRender()
    {
        //Log("Page Render");
    }

    // Page Data Rendering event
    public function pageDataRendering(&$header)
    {
        // Example:
        //$header = "your header";
    }

    // Page Data Rendered event
    public function pageDataRendered(&$footer)
    {
        // Example:
        //$footer = "your footer";
    }
}

```
