========================
Adding A Quality Setting
========================

This section will show you how to add a custom setting for your plugin. We will 
add a setting to choose the maximum quality we want to play from videobb.

The only quality values I have seen from the site are '240p' and '480p' but we
will also add a 'Maximum' value which will just play whatever the maximum 
quality is in case there are higher resolutions.

.. seealso::

    For more information on how plugin settings work please see 
    :class:`~urlresolver.plugnplay.interfaces.PluginSettings`
    
Settings XML
============

First off we need to tell XBMC what settings we want for our plugin. This is 
done by implementing :class:`~urlresolver.plugnplay.interfaces.PluginSettings` 
and overriding the 
:meth:`~urlresolver.plugnplay.interfaces.PluginSettings.get_settings_xml`
method.

We have already said we are implementing 
:class:`~urlresolver.plugnplay.interfaces.PluginSettings` so lets add the 
following code to override the method:

.. code-block:: python
    :linenos:
    
    def get_settings_xml(self):
        xml = PluginSettings.get_settings_xml(self)
        xml += '<setting label="Highest Quality" id="MyVideobbResolver_q" '
        xml += 'type="enum" values="240p|480p|Maximum" default="2" />\n'
        return xml
        
* *Line 2* - This adds the default settings (currently only 'priority')
* *Line 3-4* - This is a valid ``<setting \>`` tag that you would put in 
  ``resources/settings.xml`` of a addon. Here we make an enum which will
  return 0, 1 or 2 depending which value is set. By default it will return
  2 which corresponds to 'Maximum'. The setting id contains the class name
  of your plugin followed by an underscore and the name you want to refer to 
  this setting by. We will retrieve this setting with the name 'q'.
  
That was easy enough! When the module is loaded, the ``resources/settings.xml`` 
for ``script.module.urlresolver`` is rebuilt including each plugins settings.

Addons using ``script.module.urlresolver`` can display the plugin settings
for editing by calling :func:`urlresolver.display_settings`.

Go to 't0mm0 test addon' in XBMC and select 'resolver settings' and you should
see your new setting in the 'myvideobb' category. Now if you look at the
contents of ``script.module.urlresolver/resources/settings.xml`` you should see
the generated file with the definition of your new setting in it.


Using the Setting
=================

It isn't much use having a setting if we don't use it! So lets add some code 
to use it to determine what stream to play.

Change the last section of code in ``get_media_url()`` to the following:

.. code-block:: python
    :linenos:

        #find highest quality URL
        max_res = [240, 480, 99999][int(self.get_setting('q'))]
        r = re.finditer('"l".*?:.*?"(.+?)".+?"u".*?:.*?"(.+?)"', json)
        chosen_res = 0
        stream_url = False
        if r:
            for match in r:
                res, url = match.groups()
                res = int(res.strip('p'))
                if res > chosen_res and res <= max_res:
                    stream_url = url.decode('base-64')
                    chosen_res = res
        else:
            common.addon.log_error('myvideobb: stream url not found')
            return False

Note that we only had to add oneline and change one other:

* *Line 2* - This is a completely new line. It reads the setting and translates
  it into a value usable as a test for the maximum resolution.
  
  The first part ``[240, 480, 99999]`` sets up a list of values that we want
  to choose from. The second part ``[int(self.get_setting('q'))]`` gets the
  setting from XBMC, which will be  0, 1 or 2 and uses it as an index of the 
  list we set up in the first part of the line. We use ``int()`` because all
  settings are returned by XBMC as a string and a list index must be an int.
  
  It might be simpler to think of this line as the following equivalent code
  (comments assume that the setting has not been changed from its default value
  of 'Maximum')::
  
    options = [240, 480, 99999]
    q = self.get_setting('q') # q is now '2' (a str)
    q = int(q) # q is now 2 (an int)
    max_res = options[q] # max_res is now 99999 (an int)
    
* *Line 10* - this line has been changed to add the condition 
  ``and res <= max_res``. This just makes sure that the stream URL is only 
  selected if it is the highest quality URL that is below or equal to the 
  quality setting.
  
Now try changing the the settings and visiting the test links you created in
't0mm0 test addon' in XBMC. You should see (by pressing 'o' once the video is
playing) that the correct quality is chosen.

Don't forget that not all streams have all qualities available. If there is no
version with a low enough quality the plugin will not resolve the URL, but if 
there is not a high enough quality version you will get the highest available.
