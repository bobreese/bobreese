# FPP status page stuck on loading skeleton (master 10.x-2147-g2ac813fce)

## Symptom
After updating PiMaster from `a21091157` to `2ac813fce` on 2026-10-09, the
Status page sits on the grey loading placeholders forever. The header still
shows Idle, CPU temp and the IP address. fppd itself is healthy: the update
log shows `rc=0`, and fppd.log shows a clean start and main loop.

## Root cause
Upstream commit `031d0c71` ("fix(config): Encode server vars as JS strings")
added a comment inside the inline `<script>` block in `www/config.php` that
contains the literal text `</script>`:

    // escapes keep the value identical while also defeating the
    // literal </script> sequence for the HTML parser.

The HTML parser ends the script element at that text, even inside a JS
comment. The rest of the block, including `var settings = ...`, gets
dropped, and the browser throws `SyntaxError: Unexpected end of input`.
Every page that includes config.php then fails with
`ReferenceError: settings is not defined`. On index.php, `PageSetup()` throws
before `GetFPPDmode()` and `OnSystemStatusChange(GetFPPStatus)` run, so
nothing removes the `is-loading` class.

Reproduced with the PHP dev server and headless Chromium: the error appears
at index.php line 55, column 23 on `2ac813fce` and is gone with the patch.

## Fix
`config-php-script-tag.patch` rewords the comment so it no longer contains
`</script>`.

To apply it on the Pi now (until upstream fixes it):

    sudo sed -i 's|literal </script> sequence|literal closing-script-tag sequence|' /opt/fpp/www/config.php

Then hard-refresh the browser (Ctrl+F5). The next `fpp-update` may conflict
with, or overwrite, this local edit. If upstream has fixed it by then, run
`git -C /opt/fpp checkout www/config.php` first.
