# Implementing Auto-Login for ERP Systems

This document provides instructions for implementing auto-login functionality across all subsystems of the Enterprise Resource Planning (ERP) system.

## Steps for Implementation

For each subsystem in the ERP, follow these steps:

1. Open PHPMaker for the respective subsystem.
2. Navigate to "Server Side Events".
3. Go to "Other > Login Page".
4. Locate the "Page Load" event.
5. Replace the existing content of the `Page_Load()` function with the following script:

```php
// Page Load event
function Page_Load()
{
    global $Security, $Language;
    // Check for auto-login
    if (!$Security->isLoggedIn()) {
        $username = null;
        $subsystems = [
            'SYSTEM1',
            'SYSTEM2',
            'SYSTEM3',
            'SYSTEM4',
        ];
        foreach ($subsystems as $subsystem) {
            $userNameKey = "{$subsystem}_Status_UserName";
            $userIdKey = "{$subsystem}_Status_UserID";
            if (isset($_SESSION[$userNameKey]) && isset($_SESSION[$userIdKey])) {
                $username = $_SESSION[$userNameKey];
                break;
            }
        }
        if ($username) {
            // Validate user in UAC database
            $rs = ExecuteRow("SELECT * FROM users WHERE username = '" . AdjustSql($username) . "'", "erp_uac");
            if ($rs) {
                $Security->loginUser(
                    $rs['username'],
                    $rs['user_id'],
                    $rs['reports_to_user_id'],
                    $rs['user_levels'],
                    $rs['user_id']
                );
                // Additional actions after successful login
                $this->Username->setFormValue($username);
                $this->Password->setFormValue($rs['password_hash']);
                $this->LoginType->setFormValue("a"); // Auto login
                // Check if writeAuditTrailOnLogin method exists before calling it
                if (method_exists($this, 'writeAuditTrailOnLogin')) {
                    $this->writeAuditTrailOnLogin();
                }
                $this->userLoggedIn($username);
                $lastUrl = $Security->lastUrl();
                if ($lastUrl == "") {
                    $lastUrl = "index";
                }
                $this->terminate($lastUrl); // Redirect to last accessed URL
            } else {
                $this->setFailureMessage($Language->phrase("AutoLoginFailed"));
            }
        }
    }
}
```

## Important Notes

1. Ensure that the `$subsystems` array includes all the relevant subsystems for your  ERP implementation.

2. For the User Access Control (UAC) system:
   - Replace `"erp_uac"` with `"DB"` in the database query:
     ```php
     $rs = ExecuteRow("SELECT * FROM users WHERE username = '" . AdjustSql($username) . "'", "DB");
     ```
   This ensures that the UAC system uses its main database for authentication instead of a linked table.

3. Make sure that the database connection named "DB" is properly configured in your PHPMaker settings for each subsystem.

4. Test the auto-login functionality thoroughly across all subsystems to ensure it works correctly.

5. If any subsystem doesn't have the `writeAuditTrailOnLogin()` method, the code will skip it without causing errors.

## Troubleshooting

- If auto-login fails, check the session variables and database connections.
- Ensure that the user table structure in the UAC database matches the fields being accessed in the script.
- Verify that the `$Security` object and its methods are available and functioning correctly in each subsystem.

