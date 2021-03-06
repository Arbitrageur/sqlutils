# The sqlutils Package

The `sqlutils` package provides a set of utility functions to help manage a library of structured query language (SQL) files. The package can be installed from Github using the `devtools` package.

```R
devtools::install_github('jbryer/sqlutils')
```

The `sqlutils` package provides functions to document, cache, and execute SQL queries. The location of the SQL files is determined by the `sqlPaths()` function. This function behaves in a manner consistent with the `.libPaths()` function.
By default, a single path will be defined being the `data` directory where the `sqlutils` package is installed.

	> sqlPaths()
	[1] "/Users/jbryer/R/sqlutils/data"

Additional search paths can be added using `sqlPaths('/Path/To/SQL/Files')`. By convention, `sqlutils` will work with any plain text files with a `.sql` file extention in any of the directories returned from `sqlPaths()`. In the case of multiple files with the same name, first one wins.

In addition to working with a library (directory) of SQL files, `sqlutils` recognizes `roxygen2` style documentation. The `StudentsInRange` script (located in the `data` directory of the installed package), exemplifies how to create a SQL query with two parameters as well as how to define those parameters and provide default values. Default values are used when the user fails to supply values within the `execQuery` or `cacheQuery` functions (described in detail bellow). The available documenations tages are:

* @param *paramName* - This provides a description of the parameter.
* @default *paramName* - This defines the default value. This can be any valid R statement.
* @return *columnName* - Provides documentation for any returned columns. 

The contents of the `StudentsInRange` query follows:

	#' Students enrolled within the given date range.
	#' 
	#' @param startDate the start of the date range to return students.
	#' @default startDate format(Sys.Date(), '%Y-01-01')
	#' @param endDate the end of the date range to return students.
	#' @default endDate format(Sys.Date(), '%Y-%m-%d')
	#' @return CreatedDate the date the row was added to the warehouse data.
	#' @return StudentId the student id.
	SELECT * 
	FROM students 
	WHERE CreatedDate >= ':startDate:' AND CreatedDate <= ':endDate:'

It should be noted that parameters are replaced just before executing the query and must be contained with a pair of colons (:) and be valid R object names (i.e. not start with a number, contain spaces, or special characters).

We can now retrieve the documentation from within R using the `sqldoc` command.

	> sqldoc('StudentsInRange')
	Students enrolled within the given date range.
	Parameters:
	     param                                            desc                        default default.val
	 startDate the start of the date range to return students. format(Sys.Date(), '%Y-01-01')  2012-01-01
	   endDate   the end of the date range to return students. format(Sys.Date(), '%Y-%m-%d')  2012-11-19
	Returns (note that this list may not be complete):
	    variable                                              desc
	 CreatedDate the date the row was added to the warehouse data.
	   StudentId                                   the student id.

The required parameters can also be retrieved using the `getParameters` function.

	> getParameters('StudentsInRange')
	[1] "startDate" "endDate"

In the case there are no parameters, an empty character vector is returned.

	> getParameters('StudentSummary')
    character(0)

A list of all available queries is returned using the `getQueries()` function.

	> getQueries()
	 [1] "StudentsInRange" "StudentSummary" 

There are two functions available to execute queries, `execQuery` and `cacheQuery`. The former will send the SQL query to the database upon every execution. The latter however, maintains a local cached version (as a CSV or Rda file) of the resulting data frame. Specifically, the function creates a unique filename based upon the query name and parameters (see `getCacheFilename` function; this can also be overwritten using the `filename` parameter). If that file exists in specified directory (the current working directory by default), then it reads the file from disk and returns that. If the file does not exist, then `execQuery` is called, the result data frame saved to disk, and then the data frame is returned. The following complete example loads the `students` data frame from the `retention` package, saves it to a SQLite database, and executes the two included queries.

	> require(RSQLite)
	> sqlfile <- paste(system.file(package='sqlutils'), '/db/students.db', sep='')
	> m <- dbDriver("SQLite")
	> conn <- dbConnect(m, dbname=sqlfile)
	> q1 <- execQuery('StudentSummary', connection=conn)
	> head(q1)
	  CreatedDate count
	1  2002-07-15  8365
	2  2002-08-15  8251
	3  2002-09-15  8259
	4  2002-10-15  8258
	5  2002-11-15  8151
	6  2002-12-15  8415

### Supported databases

The `sqlutils` package supports database access using the [`RODBC`](http://cran.r-project.org/web/packages/RODBC/index.html), [`RSQLite`](http://cran.r-project.org/web/packages/RSQLite/index.html), [`RPostgreSQL`](http://cran.r-project.org/web/packages/RPostgreSQL/index.html), and [`RMySQL`](http://cran.r-project.org/web/packages/RMySQL/index.html) packages using an S3 generic function call called `sqlexec` based upon the class of the `connection` parameter. For example, create a new database connection for connections of class `foo`, the following provides the skeleton of the function to implement:

```R
sqlexec.foo <- function(connection, sql, ...) {
	#Database implementation here.
	#The ... will be passed through from the execQuery call. 
}
```	
