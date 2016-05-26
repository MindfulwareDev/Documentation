#Why to use a using

###According to MSDN, Using statements "[Provides a convenient syntax that ensures the correct use of IDisposable objects.](https://msdn.microsoft.com/en-us/library/yh598w02.aspx)"

* An [IDisposable object]("https://msdn.microsoft.com/en-us/library/system.idisposable.aspx") is "a mechanism for releasing unmanaged resources"
    - The garbage collector automatically releases the memory allocated to the managed object when that object is no longer used, but it is not possible to predict when the garbage collection will occur.
    - Use of the [Dispose](https://msdn.microsoft.com/en-us/library/system.idisposable.dispose.aspx) method of the IDisposable interface will free the resources being used by this IDisposable.
* A Using statement will cleanly handle any errors that occur while the IDisposable is in use, and will dispose all of the assets that the object used and allow the garbage collector to take care of them.
    * C#
    ```csharp
    using (StreamReader stream = new StreamReader("example.txt")){
        //Use the object here
        //note: stream cannot be referenced outside of this region    
    }
    ```
    * Visual Basic
    ```vb
    Using stream As New StreamReader("example.txt")
        //Use the object here
        //note: stream cannot be referenced outside of this region
    End Using
    ```