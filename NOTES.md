# NOTES

- packages under `internal` cannot be imported by code outside of our project

- Cool structure:
  - The cmd directory will contain the application-specific code for the executable applications in the project. For now our project will have just one executable application — the web application — which will live under the cmd/web directory
  - The internal directory will contain the ancillary non-application-specific code used in the project. We’ll use it to hold potentially reusable code like validation helpers and the SQL database models for the project.
  - The ui directory will contain the user-interface assets used by the web application. Specifically, the ui/html directory will contain HTML templates, and the ui/static directory will contain static files (like CSS and images)

## File server features and functions

Go’s http.FileServer handler has a few really nice features that are worth mentioning:

- It sanitizes all request paths by running them through the path.Clean() function before
  searching for a file. This removes any . and .. elements from the URL path, which helps
  to stop directory traversal attacks. This feature is particularly useful if you’re using the
  fileserver in conjunction with a router that doesn’t automatically sanitize URL paths.

- **Range requests** are fully supported. This is great if your application is serving large files
  and you want to support resumable downloads.

- Last-Modified and If-Modified-Since headers are transparently supported. If a
  file hasn’t changed since the user last requested it, then http.FileServer will send a
  304 Not Modified status code instead of the file itself. This helps reduce latency and
  processing overhead for both the client and server

- Content-Type is automatically set from the file extension using the
  mime.TypeByExtension() function. Custom extensions and
  content types using the mime.AddExtensionType()

- `http.FileServer` probably won’t be reading these files from
  disk once the application is up-and-running. Both Windows and Unix-based operating
  systems cache recently-used files in RAM, so (for frequently-served files at least) it’s likely
  that `http.FileServer` will be serving them from RAM rather than making the relatively slow
  round-trip to your hard disk

### Single file serving

- `http.ServeFile()` does not automatically sanitize the file path. If you’re
  constructing a file path from untrusted user input, to avoid directory traversal attacks
  you must sanitize the input with `filepath.Clean()` before using it

### Disabling directory listings

Create a custom implementation of `http.FileSystem`, and have it return an `os.ErrNotExist` error for any directories

```go

fileServer := http.FileServer(http.Dir("./ui/static/"))
mux.Handle("GET /static/", http.StripPrefix("/static", neuter(fileServer)))

func neuter(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if strings.HasSuffix(r.URL.Path, "/") {
			http.NotFound(w, r)
			return
		}

		next.ServeHTTP(w, r)
	})
}

```

- Go web application is a chain of `ServeHTTP()` methods being called one after another

### Requests are handled concurrently

- *All incoming HTTP requests are served in their own goroutine*
- Be aware of (and protect against) **race conditions** when accessing shared resources from your handlers.
