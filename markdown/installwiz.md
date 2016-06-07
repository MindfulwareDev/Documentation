# InstallWiz Technical Process

* Check the Database (build_system.Install table) for any records where the 'Started' column is null and the build was listed as a success.
    * Change any found records to read 'Underway'
* Gather Install parameters from the database records, including:
    * Build Number
    * 64 or 32 bit install
    * Abbreviation of the company
    * IP Address of the Target Server
    * IP Address of the App Server
    * Database name
    * Role
    * Install ID
    * Old Build Number
    * Email Addresses to notify
* Map the Target server to the P Drive
* Delete the previous install from the server
* Copy the new build from the ABI server to the Target server
* Create and run SILENT_INSTALL.BAT on the target server
* Create SILENT_INSTALL.INI and move it to the Target server
* Run setup.exe on the Target server via the psExec command, with the parameter ('/UNATTENDED=SILENT_INSTALL.INI') to let the program know to run in silent mode.
* Delete the SILENT_INSTALL.BAT and SILENT_INSTALL.INI files from the Target server
* Update the CossEnterpriseSuite\web.config file on the DMZ Server with the new StateDriver information from the SessionState
* Un-Map the P drive
* Update the Database record for this install to 'Completed'
* Email all associated accounts that the installation has completed

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Runtime.InteropServices;
using System.Data.SqlClient;
using System.IO;
using System.Net.Mail;
using System.Text.RegularExpressions;

namespace InstallWiz
{
    class Program
    {
        static public void ReplaceInFile(string filePath, string searchText, string replaceText)
        {
            StreamReader reader = new StreamReader(filePath);
            string content = reader.ReadToEnd();
            reader.Close();

            content = Regex.Replace(content, searchText, replaceText);

            StreamWriter writer = new StreamWriter(filePath);
            writer.Write(content);
            writer.Close();
        }

        public class Folders
        {
            public string Source { get; private set; }
            public string Target { get; private set; }

            public Folders(string source, string target)
            {
                Source = source;
                Target = target;
            }
        }
        public static void CopyDirectory(string source, string target)
        {
            try
            {
                var stack = new Stack<Folders>();
                stack.Push(new Folders(source, target));

                while (stack.Count > 0)
                {
                    var folders = stack.Pop();
                    Directory.CreateDirectory(folders.Target);
                    foreach (var file in Directory.GetFiles(folders.Source, "*.*"))
                    {
                        string targetFile = Path.Combine(folders.Target, Path.GetFileName(file));
                        if (File.Exists(targetFile)) File.Delete(targetFile);
                        File.Copy(file, targetFile);
                    }

                    foreach (var folder in Directory.GetDirectories(folders.Source))
                    {
                        stack.Push(new Folders(folder, Path.Combine(folders.Target, Path.GetFileName(folder))));
                    }
                }
            }
            catch (Exception ex)
            {

                throw new System.Exception("Copy Failed!  " + ex.Message);
            }
        }

        static void WriteLog(string p_strFileName, string p_strText, Boolean p_blnAppendFile)
        {
            try
            {
                if (p_strFileName.Length > 0)
                {
                    Stream l_LogFile;

                    if (p_blnAppendFile == false)
                    {
                        System.IO.File.Delete(p_strFileName);
                        l_LogFile = File.Open(p_strFileName, FileMode.Create, FileAccess.ReadWrite, FileShare.None);
                    }
                    else
                    {
                        l_LogFile = File.Open(p_strFileName, FileMode.Append, FileAccess.Write, FileShare.None);
                    }

                    StreamWriter l_LogFileWriter = new StreamWriter(l_LogFile);
                    l_LogFileWriter.Write(DateTime.Now.ToString() + " - " + p_strText + "\r\n");
                    l_LogFileWriter.Close();
                }
            }
            catch (System.Exception ex)
            // for debugging purposes only
            { string message1 = ex.Message; }
        }

        static void Main(string[] args)
        {
            string l_strBuild = string.Empty;
            string l_strCompany = string.Empty;
            string l_strServer = string.Empty;
            string l_strAppServer = string.Empty;
            string l_strNotify = string.Empty;
            string l_strDatabase = string.Empty;
            string l_strBuildName = string.Empty;
            string l_strBuildServer = @"\\dca-svm-inf.corp.ipipenet.com\ABI\Builds";
            // string l_strDatabaseServer = string.Empty;
            string l_strResponse = string.Empty;
            string l_strParams = string.Empty;
            string l_strCommand = string.Empty;
            string l_strRole = string.Empty;
            string l_strLinkIP = string.Empty;
            string l_strInstallID = string.Empty;
            bool l_bln64bit = false;
            bool l_bln3Tier = false;
            string[] l_astrEmailAddress;
            char[] delimiterChars = { ' ', ',', ';' };
            string l_strVersion = System.Reflection.Assembly.GetExecutingAssembly().GetName().Version.ToString();
            string l_strLog = @"E:\InstallWiz\Logs\";

            try
            {
                SqlConnection l_objCon = new SqlConnection("Data Source=dc0dvtdbs01.corp.ipipenet.com;Initial Catalog=build_system;User Id=autobuild;password=coss");
                SqlConnection l_objCon2 = new SqlConnection("Data Source=dc0dvtdbs01.corp.ipipenet.com;Initial Catalog=build_system;User Id=autobuild;password=coss");
//              SqlConnection l_objCon = new SqlConnection("Data Source=dwdb420v.dv.ipipenet.com;Initial Catalog=build_system;User Id=autobuild;password=coss");
//              SqlConnection l_objCon2 = new SqlConnection("Data Source=dwdb420v.dv.ipipenet.com;Initial Catalog=build_system;User Id=autobuild;password=coss");
                l_objCon.Open();
                l_objCon2.Open();

                SqlCommand l_objCmd = new SqlCommand("SELECT I.InstallID, I.Server, I.AppServer, S.Server_IP, I.Build, B.Build64bit, B.CompanyAbbr, I.DatabaseName, I.Notify, I.Notes, S.Role, S.Version, S.BuildNumber as OldBuild FROM [Install] I INNER JOIN [Build] B ON I.Build = B.BuildNumber INNER JOIN [Servers] S ON I.Server = S.DNS WHERE I.Started IS NULL AND B.Status like '%Success%'", l_objCon);
                SqlDataReader l_objRdr = l_objCmd.ExecuteReader();
                while (l_objRdr.Read())
                {
                    l_strBuild = l_objRdr["Build"].ToString();
                    l_bln64bit = Convert.ToBoolean(l_objRdr["Build64bit"].ToString());
                    l_strCompany = l_objRdr["CompanyAbbr"].ToString();
                    l_strServer = l_objRdr["Server_IP"].ToString();
                    l_strDatabase = l_objRdr["DatabaseName"].ToString();
                    l_strRole = l_objRdr["Role"].ToString();
                    l_strInstallID = l_objRdr["InstallID"].ToString();

                    if (l_bln64bit)
                        l_strBuildName = l_strCompany + "-" + l_strBuild + "-Connected(x64)";
                    else
                        l_strBuildName = l_strCompany + "-" + l_strBuild + "-Connected(x86)";

                    l_strLinkIP = l_objRdr["Server"].ToString();
                    l_strAppServer = l_objRdr["AppServer"].ToString();

                    // Check if 3 Tier install
                    if (l_strAppServer == string.Empty)
                        l_bln3Tier = false;
                    else
                        l_bln3Tier = true;

                    // Determine if there are multiple email addresses separated by comma or semi-colon and parse as needed
                    l_strNotify = l_objRdr["Notify"].ToString();
                    // first strip out any spaces
                    l_strNotify = l_strNotify.Replace(" ", "");
                    l_astrEmailAddress = l_strNotify.Split(delimiterChars);

                    // Update the Install table that the processing of the record has begun
                    new SqlCommand("UPDATE Install set Started='" + System.DateTime.Now + "', Status='Underway' where InstallID = '" + l_strInstallID + "'", l_objCon2).ExecuteNonQuery();

                    l_strLog = l_strLog + l_strInstallID + "-" + l_strCompany + ".log";
                    WriteLog(l_strLog, l_strCompany + " Installation started - Version: " + l_strVersion, true);
                    WriteLog(l_strLog, "  Build: " + l_strBuild, true);
                    WriteLog(l_strLog, "  Target Server: " + l_strServer, true);
                    WriteLog(l_strLog, "  App Server: " + l_strAppServer, true);

                    // delete the temporary network drive to the target server in case it still exists from a previous attempt
                    System.Diagnostics.Process.Start("net.exe", @"USE P: /DELETE");

                    // wait for the new share to delete before continuing
                    System.Threading.Thread.Sleep(20000);

                    WriteLog(l_strLog, "Mapping P drive to the Target Server", true);
                    // map a temporary network drive to the target server
                    l_strCommand = @"USE P: \\" + l_strServer + @"\e$ /USER:dv\dappsvc400 Vanilla100!";
                    
                    System.Diagnostics.Process stdin3;
                    stdin3 = System.Diagnostics.Process.Start("net.exe", l_strCommand);
                    stdin3.WaitForExit();

                    // wait for the new share to exists before continuing
                    System.Threading.Thread.Sleep(20000);

                    // Check that the P drive is created successfully
                    System.IO.DirectoryInfo di = new System.IO.DirectoryInfo("P:");
                    if (di.Exists == false)
                    {
                        WriteLog(l_strLog, "ERROR: Unable to map drive on target server.  Ensure the Server is turned on.", true);
                        throw new System.Exception("Unable to map drive on target server.  Ensure the Server is turned on.");
                    }
                    
                    // Delete the previous installs
                    WriteLog(l_strLog, "Delete Previous Install", true);
                    try
                    {
                        if (l_bln64bit)
                            Directory.Delete(@"P:\Installs\" + l_strCompany + "-" + l_objRdr["OldBuild"].ToString() + "-Connected(x64)", true);
                        else
                            Directory.Delete(@"P:\Installs\" + l_strCompany + "-" + l_objRdr["OldBuild"].ToString() + "-Connected(x86)", true);
                    }
                    catch { }

                    // Establish psExec parameters
                    string l_strProgram = "psExec";
                    l_strParams = @"\\" + l_strServer + @" -u dv\dappsvc400 -p Vanilla100! -e ";

                    if (l_strRole != "DEV")
                    {
                        WriteLog(l_strLog, "Copy Build to Target server", true);
                        // Copy the build via this machine since build share can't be mapped from all servers
                        CopyDirectory( l_strBuildServer + @"\" + l_strCompany + @"\" + l_strBuildName, @"P:\Installs\" + l_strBuildName);
                    }
                    else
                    {
                        // Create batch file to copy to the target server which will copy down the build 
                        WriteLog(l_strLog, "Creating batch file to copy build", true);
                        System.IO.StreamWriter l_oBF = new System.IO.StreamWriter(@".\SILENT_INSTALL.BAT", false);
                        l_oBF.WriteLine("net USE P: " + l_strBuildServer + @" /USER:dv\dappsvc400 Vanilla100!");
                        l_oBF.WriteLine("xcopy \"P:\\" + l_strCompany + @"\" + l_strBuildName + "\" " + "\"E:\\Installs\\" + l_strBuildName + "\" /Y /I /S /E /Q");

                        l_oBF.Close();

                        File.Copy(@".\SILENT_INSTALL.BAT", @"P:\SILENT_INSTALL.BAT", true);

                        // Run batch file on target server
                        l_strCommand = "E:\\SILENT_INSTALL.BAT";
                        System.Diagnostics.ProcessStartInfo psi = new System.Diagnostics.ProcessStartInfo(l_strProgram);
                        psi.RedirectStandardOutput = true;
                        psi.WindowStyle = System.Diagnostics.ProcessWindowStyle.Hidden;
                        psi.UseShellExecute = false;
                        System.Diagnostics.Process stdin;
                        psi.Arguments = l_strParams + " " + l_strCommand;
                        stdin = System.Diagnostics.Process.Start(psi);
                        System.IO.StreamReader myOutput = stdin.StandardOutput;
                        l_strResponse = myOutput.ReadToEnd();
                        stdin.WaitForExit(0);
                        // Check for error codes from process
                        if (stdin.ExitCode != 0)
                        {
                            throw new System.Exception("Bad Return Code: " + l_strProgram + " " + l_strParams + "\n-->" + l_strResponse);
                        }
                    }
                    // Create SILENT_INSTALL.INI file
                    WriteLog(l_strLog, "Create SILENT_INSTALL.INI file", true);
                    File.Delete(@".\SILENT_INSTALL.INI");
                    System.IO.StreamWriter l_oSI = new System.IO.StreamWriter(@".\SILENT_INSTALL.INI", false);
                    l_oSI.WriteLine("[UPGRADE]");
                    //l_oSI.WriteLine("DBSVR_ADDR=" + l_strDatabaseServer);
                    //l_oSI.WriteLine("DB_NAME=" + l_strDatabase);
                    //l_oSI.WriteLine("DB_LOGIN_NAME=cossnet");
                    //l_oSI.WriteLine("DB_LOGIN_PASSWORD=cossnet");
                    //l_oSI.WriteLine("SA_LOGIN=Autobuild");
                    //l_oSI.WriteLine("SA_PASSWORD1=coss");

                    l_oSI.Close();
                    WriteLog(l_strLog, "Copy INI file to Target server", true);
                    File.Copy(@".\SILENT_INSTALL.INI", @"P:\Installs\" + l_strBuildName + @"\SILENT_INSTALL.INI", true);

                    // Run setup.exe on target server
                    WriteLog(l_strLog, "Run Install on Target Server", true);
                    l_strCommand = @"E:\Installs\" + l_strBuildName + @"\setup.exe /UNATTENDED=SILENT_INSTALL.INI";
                    System.Diagnostics.ProcessStartInfo psi2 = new System.Diagnostics.ProcessStartInfo(l_strProgram);
                    psi2.RedirectStandardOutput = true;
                    psi2.WindowStyle = System.Diagnostics.ProcessWindowStyle.Hidden;
                    psi2.UseShellExecute = false;
                    System.Diagnostics.Process stdin2;
                    psi2.Arguments = l_strParams + " " + l_strCommand;
                    stdin2 = System.Diagnostics.Process.Start(psi2);
                    System.IO.StreamReader myOutput2 = stdin2.StandardOutput;
                    l_strResponse = myOutput2.ReadToEnd();
                    // Wait up to 30 minutes for install to complete
                    stdin2.WaitForExit(1800000);
                    // Check for error codes from process
                    if (stdin2.ExitCode != 0)
                    {
                        throw new System.Exception("Bad Return Code: " + l_strProgram + " " + l_strParams + "\n-->" + l_strResponse);
                    }

                    // Clean up the Install files
                    File.Delete(@"P:\SILENT_INSTALL.BAT");
                    File.Delete(@"P:\Installs\" + l_strBuildName + @"\SILENT_INSTALL.INI");

                    if (File.Exists(@"P:\PostInstallProcedures\PostInstallProcedures.cmd"))
                    {
                        // Run batch file on server - PostInstallProcedure.cmd
                        WriteLog(l_strLog, "Run PostInstallProcedure.cmd on Target Server", true);
                        l_strCommand = @"E:\PostInstallProcedures\PostInstallProcedures.cmd";
                        psi2 = new System.Diagnostics.ProcessStartInfo(l_strProgram);
                        psi2.RedirectStandardOutput = true;
                        psi2.WindowStyle = System.Diagnostics.ProcessWindowStyle.Hidden;
                        psi2.UseShellExecute = false;
                        psi2.Arguments = l_strParams + " " + l_strCommand;
                        stdin2 = System.Diagnostics.Process.Start(psi2);
                        myOutput2 = stdin2.StandardOutput;
                        l_strResponse = myOutput2.ReadToEnd();
                        // Wait up to 30 minutes for install to complete
                        stdin2.WaitForExit(1800000);
                        // Check for error codes from process
                        if (stdin2.ExitCode != 0)
                        {
                            throw new System.Exception("Bad Return Code: " + l_strProgram + " " + l_strParams + "\n-->" + l_strResponse);
                        }
                    }
                    // delete the temporary network drive to the target server
                    System.Diagnostics.Process.Start("net.exe", @"USE P: /DELETE");

                    // wait for the new share to deletes before continuing
                    System.Threading.Thread.Sleep(20000);

                    // Now Install to App tier if needed
                    if (l_bln3Tier)
                    {
                        WriteLog(l_strLog, "** Now installing to App Server **", true);
                        // map a temporary network drive to the target server
                        WriteLog(l_strLog, "Mapping P drive to the Target Server", true);
                        l_strCommand = @"USE P: \\" + l_strAppServer + @"\e$ /USER:dv\dappsvc400 Vanilla100!";

                        stdin3 = System.Diagnostics.Process.Start("net.exe", l_strCommand);
                        stdin3.WaitForExit();

                        // wait for the new share to exists before continuing
                        System.Threading.Thread.Sleep(20000);

                        // Delete the previous installs
                        try
                        {
                            if (l_bln64bit)
                                Directory.Delete(@"P:\Installs\" + l_strCompany + "-" + l_objRdr["OldBuild"].ToString() + "-Connected(x64)", true);
                            else
                                Directory.Delete(@"P:\Installs\" + l_strCompany + "-" + l_objRdr["OldBuild"].ToString() + "-Connected(x86)", true);
                        }
                        catch { }

                        // Establish psExec parameters
                        l_strParams = @"\\" + l_strAppServer + @" -u dv\dappsvc400 -p Vanilla100! -e ";

                        // Check that the P drive is created successfully
                        System.IO.DirectoryInfo di2 = new System.IO.DirectoryInfo("P:");
                        if (di2.Exists == false)
                        {
                            throw new System.Exception("Unable to map drive on target server.  Ensure the Server is turned on.");
                        }

                        if (l_strRole != "DEV")
                        {
                            // Copy the build via this machine since build share can't be mapped from all servers
                            WriteLog(l_strLog, "Copying Build to the Target Server", true);
                            CopyDirectory(l_strBuildServer + @"\" + l_strCompany + @"\" + l_strBuildName, @"P:\Installs\" + l_strBuildName);
                        }
                        else
                        {
                            // Create batch file to copy to the target server which will copy down the build 
                            WriteLog(l_strLog, "Creating batch file to copy build", true);
                            System.IO.StreamWriter l_oBF = new System.IO.StreamWriter(@".\SILENT_INSTALL.BAT", false);
                            l_oBF.WriteLine("net USE P: " + l_strBuildServer + @" /USER:dv\dappsvc400 Vanilla100!");
                            l_oBF.WriteLine("xcopy \"P:\\" + l_strCompany + @"\" + l_strBuildName + "\" " + "\"E:\\Installs\\" + l_strBuildName + "\" /Y /I /S /E /Q");

                            l_oBF.Close();

                            File.Copy(@".\SILENT_INSTALL.BAT", @"P:\SILENT_INSTALL.BAT", true);

                            // Run batch file on target server
                            WriteLog(l_strLog, "Run batch file on server", true);
                            l_strCommand = "E:\\SILENT_INSTALL.BAT";
                            System.Diagnostics.ProcessStartInfo psi = new System.Diagnostics.ProcessStartInfo(l_strProgram);
                            psi.RedirectStandardOutput = true;
                            psi.WindowStyle = System.Diagnostics.ProcessWindowStyle.Hidden;
                            psi.UseShellExecute = false;
                            System.Diagnostics.Process stdin;
                            psi.Arguments = l_strParams + " " + l_strCommand;
                            stdin = System.Diagnostics.Process.Start(psi);
                            System.IO.StreamReader myOutput = stdin.StandardOutput;
                            l_strResponse = myOutput.ReadToEnd();
                            stdin.WaitForExit(0);
                            // Check for error codes from process
                            if (stdin.ExitCode != 0)
                            {
                                throw new System.Exception("Bad Return Code: " + l_strProgram + " " + l_strParams + "\n-->" + l_strResponse);
                            }
                        }
                        // Create SILENT_INSTALL.INI file
                        WriteLog(l_strLog, "Create SILENT_INSTALL.INI file", true);
                        File.Delete(@".\SILENT_INSTALL.INI");
                        l_oSI = new System.IO.StreamWriter(@".\SILENT_INSTALL.INI", false);
                        l_oSI.WriteLine("[UPGRADE]");

                        l_oSI.Close();
                        File.Copy(@".\SILENT_INSTALL.INI", @"P:\Installs\" + l_strBuildName + @"\SILENT_INSTALL.INI", true);

                        // Run setup.exe on target server
                        WriteLog(l_strLog, "Run Install on Target Server", true);
                        l_strCommand = @"E:\Installs\" + l_strBuildName + @"\setup.exe /UNATTENDED=SILENT_INSTALL.INI";
                        psi2 = new System.Diagnostics.ProcessStartInfo(l_strProgram);
                        psi2.RedirectStandardOutput = true;
                        psi2.WindowStyle = System.Diagnostics.ProcessWindowStyle.Hidden;
                        psi2.UseShellExecute = false;
                        psi2.Arguments = l_strParams + " " + l_strCommand;
                        stdin2 = System.Diagnostics.Process.Start(psi2);
                        myOutput2 = stdin2.StandardOutput;
                        l_strResponse = myOutput2.ReadToEnd();
                        // Wait up to 30 minutes for install to complete
                        stdin2.WaitForExit(1800000);
                        // Check for error codes from process
                        if (stdin2.ExitCode != 0)
                        {
                            throw new System.Exception("Bad Return Code: " + l_strProgram + " " + l_strParams + "\n-->" + l_strResponse);
                        }

                        // Clean up the Install files
                        File.Delete(@"P:\SILENT_INSTALL.BAT");
                        File.Delete(@"P:\Installs\" + l_strBuildName + @"\SILENT_INSTALL.INI");

                        // Run batch file on server - PostInstallProcedure.cmd
                        if (File.Exists(@"P:\PostInstallProcedures\PostInstallProcedures.cmd"))
                        {
                            WriteLog(l_strLog, "Run PostInstallProcedure.cmd on Target Server", true);
                            l_strCommand = @"E:\PostInstallProcedures\PostInstallProcedures.cmd";
                            psi2 = new System.Diagnostics.ProcessStartInfo(l_strProgram);
                            psi2.RedirectStandardOutput = true;
                            psi2.WindowStyle = System.Diagnostics.ProcessWindowStyle.Hidden;
                            psi2.UseShellExecute = false;
                            psi2.Arguments = l_strParams + " " + l_strCommand;
                            stdin2 = System.Diagnostics.Process.Start(psi2);
                            myOutput2 = stdin2.StandardOutput;
                            l_strResponse = myOutput2.ReadToEnd();
                            // Wait up to 30 minutes for install to complete
                            stdin2.WaitForExit(1800000);
                            // Check for error codes from process
                            if (stdin2.ExitCode != 0)
                            {
                                throw new System.Exception("Bad Return Code: " + l_strProgram + " " + l_strParams + "\n-->" + l_strResponse);
                            }
                        }
                        // delete the temporary network drive to the target server
                        System.Diagnostics.Process.Start("net.exe", @"USE P: /DELETE");
                    }

                    // Update the Install table that the processing of the record has completed
                    new SqlCommand("UPDATE Install set Completed='" + System.DateTime.Now + "', Status='Complete' where InstallID = '" + l_strInstallID + "'", l_objCon2).ExecuteNonQuery();

                    // Replace to change apostrophe to double apostrophe in Notes field so SQL won't choke on it.
                    String l_strNotes = l_objRdr["Notes"].ToString().Replace("'","''");
                    
                    // Update the Server table with the install information
                    new SqlCommand("UPDATE Servers set BuildNumber='" + l_strBuild + "', DatabaseName='" + l_strDatabase + "', LastUpdate='" + System.DateTime.Now + "', Notes='" + l_strNotes + "' where Server_IP = '" + l_strServer + "'", l_objCon2).ExecuteNonQuery();

                    // send email
                    string l_strBody = "<!DOCTYPE HTML PUBLIC \"-//W3C//DTD HTML 4.0 Transitional//EN\">";
                    l_strBody += "<HTML><HEAD><META http-equiv=Content-Type content=\"text/html; charset=iso-8859-1\">";
                    l_strBody += "</HEAD><BODY><DIV><FONT face=Calibri>";
                    l_strBody += "The <b>" + l_objRdr["Version"].ToString() + " " + l_strCompany + "</b> install of build <b>" + l_strBuild + @"</b> has successfully completed.<br/><br/>";
                    l_strBody += l_strRole + @": http://" + l_strLinkIP + @"/CossEnterpriseSuite<br/><br/>";
                    l_strBody += "Notes: <b>" + l_objRdr["Notes"].ToString() + "</b>";
                    l_strBody += "</FONT></DIV></BODY></HTML>";

                    MailMessage message = new MailMessage();
                    message.From = new MailAddress("AutoBuild@ipipeline.com","ABI - Install");
                    foreach (string l_strEmailAddr in l_astrEmailAddress)
                    {
                        if (!(l_strEmailAddr == string.Empty))
                        {
                            message.To.Add(l_strEmailAddr);
                        }
                    }
                    message.CC.Add(new MailAddress("BuildServices@ipipeline.com"));
                    message.Subject = "ABI - Successful Install of " + l_strCompany + " build #" + l_strBuild + " - " + l_strRole;
                    message.IsBodyHtml = true;
                    message.Body = l_strBody;
                    SmtpClient client = new SmtpClient("mta.ipipeline.us");
                    client.Send(message);

                    WriteLog(l_strLog, "Install Succeeded!", true);

                    // only want to process first one so break out of while loop now
                    break;
                }

            }
            catch (System.Exception ex)
            {
                string l_strDesc = ex.Message;
                l_strDesc = System.Text.RegularExpressions.Regex.Replace(l_strDesc, "'", "''");
                // strip out the password when present in the error message.
                if (l_strDesc.IndexOf("-p") > 0)
                {
                    l_strDesc = l_strDesc.Substring(1, l_strDesc.IndexOf("-p"));
                }
                WriteLog(l_strLog, "Install Failed: " + l_strDesc, true);
                try
                {
                    // delete the temporary network drive to the target server
                    System.Diagnostics.Process.Start("net.exe", @"USE P: /DELETE");

                    SqlConnection l_objCon2 = new SqlConnection("Data Source=dc0dvtdbs01.corp.ipipenet.com;Initial Catalog=build_system;User Id=autobuild;password=coss");
                    l_objCon2.Open();

                    // Update the Install table that the processing of the record has failed
                    new SqlCommand("UPDATE Install set Status='Failed' where InstallID = '" + l_strInstallID + "'", l_objCon2).ExecuteNonQuery();

                    // send email
                    string l_strBody = "<!DOCTYPE HTML PUBLIC \"-//W3C//DTD HTML 4.0 Transitional//EN\">";
                    l_strBody += "<HTML><HEAD><META http-equiv=Content-Type content=\"text/html; charset=iso-8859-1\">";
                    l_strBody += "</HEAD><BODY><DIV><FONT face=Calibri>";
                    l_strBody += "ERROR: " + l_strDesc;
                    l_strBody += "</FONT></DIV></BODY></HTML>";

                    MailMessage message = new MailMessage();
                    message.From = new MailAddress("AutoBuild@ipipeline.com","ABI - Install");
                    l_astrEmailAddress = l_strNotify.Split(delimiterChars);
                    foreach (string l_strEmailAddr in l_astrEmailAddress)
                    {
                        if (!(l_strEmailAddr == string.Empty))
                        {
                            message.To.Add(l_strEmailAddr);
                        }
                    }
                    message.CC.Add(new MailAddress("BuildServices@ipipeline.com"));
                    message.Subject = "ABI - Failure of Install of " + l_strCompany + " build #" + l_strBuild + " on " + l_strServer;
                    message.IsBodyHtml = true;
                    message.Body = l_strBody;
                    SmtpClient client = new SmtpClient("mta.ipipeline.us");
                    client.Send(message);

                    // Send email notifcation that the install has failed
                    //ChangelogBroadcaster.clsSMTP smtp = new ChangelogBroadcaster.clsSMTP();
                    //smtp.server = "vmailcorp410.corp.ipipenet.com";
                    //smtp.from = "AutoBuild@ipipeline.com";
                    //smtp.subject = "FYI: Failure of Install of " + l_strCompany + " build #" + l_strBuild + " on " + l_strServer;
                    //smtp.bodyHtml = l_strDesc;
                    //// smtp.to.Add("BuildServices@ipipeline.com");
                    //smtp.to.Add("dcordon@ipipeline.com");
                    //l_astrEmailAddress = l_strNotify.Split(delimiterChars);
                    //foreach (string l_strEmailAddress in l_astrEmailAddress)
                    //{
                    //    smtp.to.Add(l_strEmailAddress);
                    //}
                    //smtp.Send();
                }
                catch
                { }
                //throw ex;
            }
        }
    }
}

``` 
