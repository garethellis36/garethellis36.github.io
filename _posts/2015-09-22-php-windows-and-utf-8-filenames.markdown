---
layout: post
title:  "PHP, Windows & UTF-8 filenames"
date:   2015-09-22 15:53:01
comments:   true
categories: php
---
I develop on Windows for applications that run on Windows servers. I am therefore well aware of a [very old PHP bug](https://bugs.php.net/bug.php?id=46990)
affecting file names containing UTF-8 characters and PHP running on Windows. I understand that this bug was ear-marked
to be addressed by the unicode support in PHP6. Of course, PHP6 was abandoned and the upcoming PHP7 does not have the 
promised unicode support.

   
# The problem #
Here is a quick example. I created a very simple form with a file input called `testing-utf8`, and an empty text file 
called `Björn.txt`. I then ran the file on PHP's
built-in server on my work Windows machine.   
   
Here is the result of doing `var_dump($_FILES)` after uploading this file in this PHP form. As you can see, Windows PHP
handles the UTF-8 character in the filename without issue.

![var_dump results](/assets/utf8-filenames-1.png)

Nor does Windows have issues with UTF-8 characters in filenames generally. Here is a screenshot from Windows Explorer of
the text file sitting in a folder quite happily:

![filesystem all OK](/assets/utf8-filenames-2.png)

But if we then take the uploaded text file and copy it using `move_uploaded_file()`, things go awry:

![filesystem ruh-roh](/assets/utf8-filenames-3.png)

Oddly, if you delete the file with the non-mangled characters, leaving only the uploaded version with the  
corrupted name, `file_exists('Björn.txt')` still returns true. Similarly, if you access the uploaded file via URL and
include the non-mangled UTF-8 character, it works fine. Essentially, PHP doesn't see that there's a problem, but Windows
does. If you try to attach the file to an email sent with PHP, the email attachment will come through with the 
corrupted version of the name. I found a function on the internet some time ago to address this particular problem,
and I must admit I can't remember where I got it from - I think it was a WordPress plugin. But here it is, as it is
in our codebase, still with the comments inserted by the original author:

{% highlight php %}
/**
 * sanitize uploaded file name
 * Fixes encoding for utf8 and non standard characters on windows.
 */
public static function sanitizeFilename($filename, $utf8 = true)
{
    if (self::seems_utf8($filename) == $utf8) {
        return $filename;
    }

    // On Windows platforms, PHP will mangle non-ASCII characters, see http://bugs.php.net/bug.php?id=47096
    if ('WIN' == substr(PHP_OS, 0, 3)) {
        if (setlocale(LC_CTYPE, 0) == 'C') { // Locale has not been set and the default is being used, according to answer by Colin Morelli at http://stackoverflow.com/questions/13788415/how-to-retrieve-the-current-windows-codepage-in-php
                    // thus, we force the locale to be explicitly set to the default system locale
                    $codepage = 'Windows-'.trim(strstr(setlocale(LC_CTYPE, ''), '.'), '.');
        } else {
            $codepage = 'Windows-'.trim(strstr(setlocale(LC_CTYPE, 0), '.'), '.');
        }
        $charset = 'UTF-8';
        if (function_exists('iconv')) {
            if (false == $utf8) {
                $filename = iconv($charset, $codepage.'//IGNORE', $filename);
            } else {
                $filename = iconv($codepage, $charset, $filename);
            }
        } elseif (function_exists('mb_convert_encoding')) {
            if (false == $utf8) {
                $filename = mb_convert_encoding($filename, $codepage, $charset);
            } else {
                $filename = mb_convert_encoding($filename, $charset, $codepage);
            }
        }
    }

    return $filename;
}
{% endhighlight %}

This also works for fixing the filename in other instances, for example if you are sending the file to the user for download
using `header()`.   

There is also an innovative workaround involving `urlencode()` and `urldecode()` presented on [this StackOverflow answer](http://stackoverflow.com/questions/1525830/how-do-i-use-filesystem-functions-in-php-using-utf-8-strings). 
The biggest issue with this for me is that on Windows you are limited to 255 characters for the full file path (not just 
the file name), and if you are working with long path names already, adding extra characters is sometimes unhelpful.
   
However, if you are writing the file to directory and you need it to actually look nice in Windows, vanilla PHP does
not present any viable solutions. For some time I struggled with this issue: our application has a feature which allows
users to export files uploaded by customers straight onto our file server, and obviously it is advantageous to be able to
keep the original filename clearly visible in Windows. Until I discovered the solution we now use, we had to make do with
users being alerted that a file had copied with corrupt characters and they had to manually correct the issue themselves.
Not great, especially given that the exporting of files from our application was intended to replace manual processes in 
the first place.   
   
# The solution #
Enter [WFIO](https://github.com/kenjiuno/php-wfio), a PHP extension which enables proper and correct interaction between
UTF-8 characters, PHP and the Windows filesystem! It's very easy to install - you simple download the appropriate DLL file, 
place it in your PHP extension directory and enable it in your PHP.ini:
{% highlight ini %}
extension=php_wfio.dll
{% endhighlight %}

Once installed, you get some new functions (such as `wfio_fopen()`) as well as a stream wrapper for use with many of PHP's
core filesystem functions. Going back to our original example using `move_uploaded_file()`, you can now do this:
{% highlight php %}
move_uploaded_file($tmp_name, "wfio://" . $path . $name);
{% endhighlight %}

This will result in the file being copied to `$path` in the filesystem, and when you navigate to that location in Windows
Explorer, the filename will display correctly.   
   
I have been using this extension for more than a year now and can wholeheartedly recommend its efficacy in a production
environment. Check out the README on Github (link above) to see all of the core functions that the WFIO wrapper augments.