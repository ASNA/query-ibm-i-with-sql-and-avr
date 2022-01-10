### Query IBM i with ASNA Visual RPG and an RPG program call

ASNA DataGate, Visual RPG's (AVR) database engine. doesn't support SQL; it is purely and a record level access engine. As an alternative, though, DataGate does support a performant RPG program call (Call/Parm). This article shows how to use AVR's program call to use SQL on the IBM i to get query results back to our app. 

### The challenge using a program call 

Using a program call to perform an SQL query with AVR is that data types returned by the program call are constrained by the limited, scalar types the of program call parameter list. It would be possible to use a data structure array to return data to the AVR from the IBM i, but creating the correct structures, populating them, and reading them, is error-prone and fiddly. 

This technique circumvents that problem by, from the RPG program, writing the query results to a file in QTEMP, and then upon returning from the program call, opening the file in QTEMP and reading its rows. It sounds a little convoluted and there are some specific steps you need to make but performance is very good--a 15-row results set is prepared and then read in the AVR program in about 300-350 milliseconds.

### The RPG program

A very simple, general-purpose RPG program executes an SQL statement passed to it. This is a lot of code--more than I hoped the task would need. However, this code provide a good boiler-plate and wouldn't take too long to copy and other modify to use with a different file. 

    Ctl-Opt Option(*srcstmt) Dftactgrp(*No) ActGrp('avrquery');

    Dcl-Pi *N;
        Sql Char(1024);
        RowsFetched Int(10);
        StatusCode Int(10);
        ErrorMsg Char(256);
        SqlError Char(24); 
    End-Pi;

    Dcl-S RowsReturned Int(10);

    ErrorMsg = '';
    SqlError = '';
    StatusCode = 0;

    ConfigureSQL();

    RowsFetched = ExecSql(Sql);
    If RowsFetched = -1; 
        StatusCode = -1;
        SqlError = 'query 1';
    EndIf; 

    Return;

    Dcl-Proc ConfigureSQL;
        Dcl-Pi *N Int(10);
        End-Pi;

        EXEC SQL
            SET OPTION COMMIT = *NONE;

        EXEC SQL
            SET OPTION NAMING = *SYS;

        return 0;            
    End-Proc;

    Dcl-Proc ExecSql;
        Dcl-Pi *N Int(10);
            Sql Char(1024); 
        End-Pi;

        Dcl-S RowsFetched Int(10);
        RowsFetched = 0; 

        If Sql = *Blanks;
            Return 0;
        EndIf;                

        EXEC SQL
            EXECUTE IMMEDIATE :Sql;
        if (SQLCOD <> 0);
            ErrorMsg = 'query SQLCOD = ' + %char(SQLCOD);
            Return -1; 
        endif;                

        EXEC SQL
            GET DIAGNOSTICS :RowsFetched = ROW_COUNT;

        Return RowsFetched;
   End-Proc;

> For production use this RPG program should probably return more diagnostic info if the SQL provided causes an issue. 

#### The SQL used

In this example, SQL statements are used.    

One creates a target work table in QTEMP:

```
CREATE TABLE qtemp/{objectId} LIKE examples/cmastnew'    
```

And the other executes a query that inserts its results into that template in QTEMP: 

```
INSERT INTO qtemp/{objectId} ')
SELECT * FROM examples/cmastnewl2 ')
WHERE concat(trim(cmname),char(cmcustno)) > ''{lastCMName}'' ')
ORDER BY concat(trim(cmname),char(cmcustno)) ')
FETCH FIRST 16 ROWS ONLY ')
```

In both cases, the SQL is created in the AVR program and substitute parameters are also replaced there. In both cases, the ready-to-execute SQL is passed to the RPG program. Not only is easier to create the SQL in the AVR program but it also means that a single RPG program can drive all of the SQL work needed. 

This technique creating unique target objects in QTEMP. The AVR `GenerateUniqueId` method show [in this GitHub repo](https://github.com/ASNA/generate-unique-ibmi-object-name) creates the necessary unique object name. 


### AVR program to create and fetch query results

```
Using System
Using System.Collections.Generic
Using System.Text
Using System.Data 
Using System.Diagnostics
Using ShortId 
Using ShortId.Configuration 
Using System.Text.RegularExpressions

BegClass Program

    BegSr Main Shared(*Yes) Access(*Public) Attributes(System.STAThread())
        DclSrParm args Type(*String) Rank(1)

        DclFld tc Type(CustomerQuery) New(15) 
        DclFld key Type(ConsoleKeyInfo) 
        DclFld sw Type(StopWatch) New() 

        tc.Query(CustomerQuery.QueryTypeEnum.ByIndexFirstPage) 
        tc.ShowPageRows()
        Console.WriteLine('Press any key to continue, or ''q'' to quite..') 
        key = Console.ReadKey(*True) 

        DoWhile key.Key <> ConsoleKey.Q 
            tc.Query(CustomerQuery.QueryTypeEnum.ByIndexNextPage)
            tc.ShowPageRows()
            Console.WriteLine('Press any key to continue, or ''q'' to quite..') 
            key = Console.ReadKey(*True) 
        EndDo 

        tc.CloseData()
    EndSr
EndClass


BegClass CustomerQuery
    DclDB DGDB DBName("*PUBLIC/Cypress")

    DclDiskFile  WebTempFile +
        Type(*Input) +
        Org(*Arrival) +
        File("Examples/CMastNewL2") +
        DB(DGDB) +
        ImpOpen(*No) 

    DclMemoryFile WebTempFileMF +
          DBDesc("*Public/Cypress") +
          FileDesc("Examples/CMastNewL2") +
          ImpOpen(*Yes) +
          RnmFmt(RMemFile)

    DclFld LastCMName Like(CMName) Access(*Public) 
    DclFld LastCMCustNo Like(CMCustNo) Access(*Public) 

    DclFld MoreRowsToRead Type(*Boolean) Access(*Public) 
    DclFld MsToPopulate Type(*Integer4) Access(*Public) 
    DclFld Customers Type(List(*Of CustomerModel)) Access(*Public) 

    DclFld RowsToShow Type(*Integer4) 
    DclFld UniqueObjectId Type(*Char) Len(10) 

    DclConst SUCCESS Value(0)
    
    BegEnum QueryTypeEnum Access(*Public) 
        DclEnumFld ByIndexFirstPage 
        DclEnumFld ByIndexNextPage 
        DclEnumFld NameStartsWith    
    EndEnum 

    BegConstructor Access(*Public) 
        DclSrParm RowsToShow Type(*Integer4) 

        *This.RowsToShow = RowsToShow
        Connect DGDB
    EndConstructor 

    BegSr CloseData Access(*Public) 
        Disconnect DGDB 
    EndSr

    BegSr Query Access(*Public) 
        DclSrParm QueryType Type(QueryTypeEnum) 

        If QueryType = QueryTypeEnum.ByIndexFirstPage
            *This.LastCMName = String.Empty
            *This.LastCMCustNo = 0 
        EndIf 

        DclFld sw Type(StopWatch) New() 

        sw.Start()
        If QueryDispatch(QueryType) = SUCCESS
            CollectPageRows()
            sw.Stop()
            *This.MsToPopulate = sw.ElapsedMilliseconds
        EndIf 
    EndSr

    BegFunc QueryDispatch Type(*Integer4) 
        DclSrParm QueryType Type(QueryTypeEnum) 

        DclFld Status Type(*Integer4) 

        Status = CreateWorkTable()
        If Status <> SUCCESS
            Throw *New ApplicationException('An error occurred creating working table.')
        EndIf 

        Status = GetQueryByPage(QueryType)
        If Status <> SUCCESS
            Throw *New ApplicationException('An error occurred performing the query.')
        EndIf 

        LeaveSr SUCCESS 
    EndFunc

    BegFunc GetQueryByPage Type(*Integer4) 
        DclSrParm QueryType Type(QueryTypeEnum) 
        
        DclFld SqlBuilder Type(StringBuilder) New()

        If QueryType = QueryTypeEnum.ByIndexFirstPage OR QueryType = QueryTypeEnum.ByIndexNextPage
            SqlBuilder.AppendLine(' INSERT INTO qtemp/{objectId} ')
            SqlBuilder.AppendLine(' SELECT * FROM examples/cmastnewl2 ')
            SqlBuilder.AppendLine(' WHERE concat(trim(cmname),char(cmcustno)) > ''{lastCMName}'' ')
            SqlBuilder.AppendLine(' ORDER BY concat(trim(cmname),char(cmcustno)) ')
            SqlBuilder.AppendLine(' FETCH FIRST {rows} ROWS ONLY ')

        // Not tested yet! 
        ElseIf QueryType = QueryTypeEnum.NameStartsWith
            SqlBuilder.AppendLine(' INSERT INTO qtemp/{objectId} ')
            SqlBuilder.AppendLine(' SELECT * FROM examples/cmastnewl2 ')
            SqlBuilder.AppendLine(' WHERE LOWER(cmname) LIKE ''{lastCMName}%'' OR concat(LOWER(trim(cmname)),char(cmcustno)) >= ''{lastCMName}'' ')
            SqlBuilder.AppendLine(' ORDER BY concat(trim(cmname),char(cmcustno)) ')
            SqlBuilder.AppendLine(' FETCH FIRST {rows} ROWS ONLY ')
        EndIf 

        SqlBuilder.Replace('{objectId}', UniqueObjectId)         
        If *This.LastCMName = ''
            SqlBuilder.Replace('{lastCMName}',  '')
        Else
            SqlBuilder.Replace('{lastCMName}',  (*This.LastCMName.Trim() + *This.LastCMCustNo.ToString()).Replace("'", "''")) 
        EndIf 
        
        // Read one more row than requested.
        SqlBuilder.Replace('{rows}', (*This.RowsToShow + 1).ToString()) 

        LeaveSr CallRPGQueryPgm(SqlBuilder.ToString()) 
    EndFunc 

    BegSr CollectPageRows
        DclFld cust Type(CustomerModel)
        DclFld i Type(*Integer4) 
        DclFld TotalRowsRead Type(*Integer4) 

        *This.Customers = *New List(*Of CustomerModel)()

        WebTempFile.FilePath = String.Format('qtemp/{0}', *This.UniqueObjectId)
        Open WebTempFile 

        TotalRowsRead = WebTempFile.RecCount
        *This.MoreRowsToRead = (TotalRowsRead > *This.RowsToShow) 

        Do FromVal(1) ToVal(*This.RowsToShow) Index(i) 
            Read WebTempFile 
            Write WebTempFileMF 

            If WebTempFile.IsEof 
                Leave 
            EndIf 

            cust = CustomerModel.GetInstanceFromDataRow(WebTempFileMF.DataSet.Tables[0])
            WebTempFileMF.DataSet.Tables[0].Clear()
            Customers.Add(cust) 
        EndDo 

        *This.LastCMName = CMName
        *This.LastCMCustNo = CMCustNo 

        Close WebTempFile
    EndSr
    
    BegSr ShowPageRows Access(*Public) 
        DclFld i Type(*Integer4) 

        Console.Clear()

        If *This.MoreRowsToRead
            Console.WriteLine('More...')            
        Else
            Console.WriteLine('Last page...')            
        EndIf

        ForEach Cust Type(CustomerModel) Collection(*This.Customers) 
            Console.WriteLine('{0} - {1}', i, cust.CMName.Trim() + '-' + cust.CMCustNo.ToString()) 
            i += 1 
        EndFor 

        Console.WriteLine('-----------------------')
        Console.WriteLine('{0}', *This.LastCMName.Trim() + *This.LastCMCustNo.ToString())
        Console.WriteLine('-----------------------')
        Console.WriteLine('---> Milliseconds: {0:#,000}', *This.MsToPopulate) 
    EndSr

    // ===================================================================    
    // Common routines - Not file-specific. 
    // ===================================================================

    BegFunc CreateWorkTable Type(*Integer4) 
        DclFld SqlBuilder Type(StringBuilder) New()

        *This.UniqueObjectId = GenerateUniqueObjectId() 

        SqlBuilder.AppendLine('CREATE TABLE qtemp/{objectId} LIKE examples/cmastnew')
        SqlBuilder.Replace('{objectId}', *This.UniqueObjectId) 
        SqlBuilder.ToString()

        LeaveSr CallRPGQueryPgm(SqlBuilder.ToString()) 
    EndFunc 

    BegFunc CallRPGQueryPgm Type(*Integer4)  
        DclSrParm Sql Type(*String) 

        // PageQuery program parm list.
        DclFld Sql1 Type(*Char) Len(1024) 
        DclFld RowsFetched Type(*Integer4) 
        DclFld StatusCode Type(*Integer4) 
        DclFld ErrorMsg Type(*Char) Len(256)
        DclFld SqlErrorMsg Type(*Char) Len(24) 

        Sql1 = Sql

        Call 'rpzimmie/pagequery' DB(DGDB) 
        DclParm Sql1
        DclParm RowsFetched  
        DclParm StatusCode 
        DclParm ErrorMsg 
        DclParm SqlErrorMsg

        If StatusCode = -1
            Console.WriteLine('Error message: {0}', StatusCode) 
            Console.WriteLine('SQL error message: {0}', SqlErrorMsg) 
            Throw *New ApplicationException('The RPG program call ended in error.')
        EndIf 

        LeaveSr 0 
    EndFunc 

    BegFunc GenerateUniqueObjectId Type(*Char) Len(10) 
        DclFld Id Type(*Char) Len(10)

        DclFld Options Type(GenerationOptions) New()
        Options.Length = 10
        // ShortId needs at least 50 unique characters. 
        ShortId.SetCharacters('ABCDEFGHIJKLMNOPQRSTUVWXYZ01234567890$#@abcdefghijklmnopqrstuvwxyz')

        Id = ShortId.Generate(Options).ToUpper()
        // Result cannot contain _, -, start with a digit
        DoWhile Id.Contains('_') OR Id.Contains('-') OR Regex.IsMatch(Id, "^\d") 
            Id = ShortId.Generate(Options).ToUpper()
        EndDo             

        LeaveSr Id
    EndFunc

EndClass
```

#### CustomerModel class

This class is generated by an AVR utility. [put repo link]

```
Using System
Using System.Text
Using System.Data
Using System.Reflection


BegClass CustomerModel Access(*Public)

    // This class was created with the CreateClassDefinition utility.
    // Do not change it by hand--regenerate it with the CreateClassDefinition utility.
    // Property names must be lowercase to correspond to AddGetInstanceFromDataRowMethod's
    // lowercase property name look up.

    DclProp cmcustno Type(System.Decimal) Access(*Public)
    DclProp cmname Type(System.String) Access(*Public)
    DclProp cmaddr1 Type(System.String) Access(*Public)
    DclProp cmcity Type(System.String) Access(*Public)
    DclProp cmstate Type(System.String) Access(*Public)
    DclProp cmcntry Type(System.String) Access(*Public)
    DclProp cmpostcode Type(System.String) Access(*Public)
    DclProp cmactive Type(System.String) Access(*Public)
    DclProp cmfax Type(System.Decimal) Access(*Public)
    DclProp cmphone Type(System.String) Access(*Public)

    BegFunc GetInstanceFromDataRow Type(CustomerModel) Access(*Public) Shared(*Yes)
        DclSrParm dt Type(DataTable)

        DclFld ColumnName Type(*String)
        DclFld obj Type(CustomerModel) New()
        DclFld dr Type(DataRow)
        DclFld i Type(*Integer4)
        DclFld pi Type(PropertyInfo)
        DclFld Value Type(*Object)

        DclFld dc Type(DataColumn)

        dr = dt.Rows[0]

        For Index(i = 0) To(dt.Columns.Count - 1)
            dc = dt.Columns[i]
            ColumnName = dt.Columns[i].ColumnName
            Value = dr.ItemArray[i]

            pi = obj.GetType().GetProperty(ColumnName.ToLower())
            If pi <> *Nothing
                pi.SetValue(obj, Convert.ChangeType(Value, pi.PropertyType), *Nothing)
            EndIf
        EndFor

        LeaveSr obj
    EndFunc

EndClass
```