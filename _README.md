# About this fork

This fork was created to investigate the slow performance of `youtube-dl` with [harbour-ytplayer](http://github.com/direc85/harbour-ytplayer) and all support for other sites than YouTube has been removed.

All bugs are considered as features, until you can reproduce them using the official release :)

## Background

First I was generally suspecting Python, because simply running `youtube-dl --version` took a long time to complete. There could not be any network delays that caused the slow execution. Another thought was that perhaps the large code base with tens of supported sited would slow the execution down. With these in the back of my head, I started my work.

As I am not very familiar with Python, I also discovered that the the `youtube-dl` binary is simply a zipped collection of `.py` files. As I tested various ways to run the scripy, I came into the conclusion that the zip file must be decompressed and compiled every single time the script is called. This adds a lot of overhead to the execution.

I found out that [Nuitka](https://www.nuitka.net) can turn Python into C++, so I gave that a try. I tested both the "full" version and the stripped down version. The command used to compile the binary on all platforms was `python3 -m nuitka --follow-imports __main__.py`.

## Removing unused code

As youtube-dl is used to only access YouTube in the project, I started removing support for everything but YouTube. This took a little trial and error, but wasn't too hard. Basically:

- Remove unnecessary extractors
- Remore references to the removed extractors from extractors.py and generic.py
- `make`

You can check the git changes for details if you are interested.

## Benchmarks

The version tested is 2020.02.16. I ran the tests on the devices using local terminal, running the command `time python3 ./youtube-dl` for the scripts and `time ./python-dl.bin` for the Nuitka compiled versions. For the unzipped Python script runs, the first run always generates the `*.pyc` files, and this run takes more time. Naturally, these first runs were discarded. With that out of the way; I ran the command five more times in a row and took the average.

My desktop computer was running up-to-date Manjaro Linux with kernel 5.5.2-1. All other devices were running Sailfish OS 3.2.1.

First, the execution times in seconds:

<table>
<tr><th>Device \ Variant</th><th>ytdl-full.py<br>zipped</th><th>ytdl-lite.py<br>zipped</th><th>ytdl-full<br>unzipped</th><th>ytdl-lite<br>unzipped</th><th>ytdl-full<br>Nuitka</th><th>ytdl-lite<br>Nuitka</th></tr>
<tr><td>AMD Ryzen 7 2700             </td><td>1.111</td><td>0.275</td><td>0.114</td><td>0.114</td><td>0.219</td><td>0.116</td></tr>
<tr><td>Sony Xperia XA2 Ultra        </td><td>6.051</td><td>1.741</td><td>1.399</td><td>0.516</td><td>1.287</td><td>0.643</tr>
<tr><td>Sony Xperia Z3 Compact Tablet</td><td>6.366</td><td>1.658</td><td>1.658</td><td>0.666</td><td>1.369</td><td>0.715</tr>
<tr><td>Jolla Phone                  </td><td>12.17</td><td>3.138</td><td>2.976</td><td>1.218</td><td>2.422</td><td>1.195</tr>
</table>

And then the same data as percentages. For each row, 100% is the largest execution time, which is the vanilla, zipped full version of `youtube-dl`.

<table>
<tr><th>Device \ Variant</th><th>ytdl-full.py<br>zipped</th><th>ytdl-lite.py<br>zipped</th><th>ytdl-full<br>unzipped</th><th>ytdl-lite<br>unzipped</th><th>ytdl-full<br>Nuitka</th><th>ytdl-lite<br>Nuitka</th></tr>
<tr><td>AMD Ryzen 7 2700             </td><td>100%</td><td>25%</td><td>10%</td><td>10%</td><td>20%</td><td>10%</td></tr>
<tr><td>Sony Xperia XA2 Ultra        </td><td>100%</td><td>29%</td><td>23%</td><td> 9%</td><td>21%</td><td>11%</td></tr>
<tr><td>Sony Xperia Z3 Compact Tablet</td><td>100%</td><td>26%</td><td>26%</td><td>10%</td><td>22%</td><td>11%</td></tr>
<tr><td>Jolla Phone                  </td><td>100%</td><td>26%</td><td>24%</td><td>10%</td><td>20%</td><td>10% </td></tr>
</table>

The desktop computer is by far the fastest, as expected. Interestingly, just uncompressing the full version gives the maximum reduction in execution time, whereas the mobile devices still benefit greatly using the uncompressed lite version.

For the mobile devices, the results are practically identical. Just uncompressing the vanilla Python zip cuts ~75% off the execution time, which is very close with using the zipped lite version. The best time was achieved with unzipped lite version, with ~90% of the execution time cut off.

Let's look at the disk space usage for all cases:

<table>
<tr><td>ytdl-full zip</td><td>ytdl-lite zip</td><td>ytdl-full unc.</td><td>ytdl-lite unc.</td><td>Nuitka full<br>armh7l / amd64</td><td>Nuitka lite<br>armh7l / amd64</td></tr>
<tr><td>1.7 MB       </td><td>0.25 MB      </td><td>5.2 MB        </td><td>1.0 MB        </td><td>26.3 MB / 38.6 MB            </td><td>4.9 MB / 7.7 MB</td></tr>
</table>
Using Nuitka to convert both full and lite versions of youtube-dl to C++ binaries resulted in good results, too. However, for mobile use cases, it is cumbersome to use as it needs to be compiled on the device, as there is no cross compilation support. It should be possible to get it running on the Sailfish OS Build Engine, but I couldn't figure out a simple way to achieve this. Another downside of using Nuitka is the size of the resulting binary. The best trade-off would be using the stripped down zip, which is the smallest, and still cuts down ~75% of the execution time. The fastest variation with 1% margin seems to be the uncompressed and stripped down version, with Nuitka binary very close behind.

I uploaded all the binaries produces by Nuitka as compiling them may be cumbersome. They have not been tested in any way (except the version string query) so please proceed with caution.

## Results

As a conclusion, I'll prefer the uncompressed lite version of youtube-dl in my project.
