# Audit Log — append-only, structured

# Format: [YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>
# Actions: WRITE, EDIT, DELETE, EXEC, QUERY, READ-EXT, VERSION, SESSION-START, SESSION-END
# Never log file contents, query result data, or credentials. Structure only.
