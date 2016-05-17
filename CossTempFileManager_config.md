Coss Temp File Manager Config
======================

* Found in the bin folder with the CossTempFileManager.exe

* Configuation file for the [CossTempFileManager](./CossTempFileManager.md)

```XML
<CossTempFileManager>
	<IntervalInSeconds>86400</IntervalInSeconds>
	<DaysToKeepFiles>28</DaysToKeepFiles>
	<LogFileName LogActivity="True">E:\CossTempFileManager\Logs\CossTempFileManager_[DATE].log</LogFileName>
    <Directories>
        <Directory [DaysToKeepFiles="28"]>E:\Logs</Directory>
    </Directories>
</CossTempFileManager>
```