# mongodumpRestoreMVP


-----------------------------------------

# Dev notes:

Mapping of the actual mongo shell command to the script command:

1. Dump all databases

python dump_restore.py config.cfg --dump --all

Mongo command run by script:

mongodump --host=localhost --port=27017 --username=admin --password=secret \  --authenticationDatabase=admin --out=/home/manjiri/dump/d0919_744

2. Dump a single database

python dump_restore.py config.cfg --dump --db testdb1

Mongo command run by script:

mongodump --host=localhost --port=27017 --username=admin --password=secret \  --authenticationDatabase=admin --out=/home/manjiri/dump/d0919_744 --db=testdb1

3. Dump specific collections from a database

python dump_restore.py config.cfg --dump --db testdb2:employees,projects

Mongo commands run (one per collection):

mongodump --host=localhost --port=27017 --username=admin --password=secret \  --authenticationDatabase=admin --out=/home/manjiri/dump/d0919_744 --db=testdb2 --collection=employees

mongodump --host=localhost --port=27017 --username=admin --password=secret \  --authenticationDatabase=admin --out=/home/manjiri/dump/d0919_744 --db=testdb2 --collection=projects


4. Restore all databases from dump

python dump_restore.py config.cfg --restore --all

Mongo command run by the script:

mongorestore --host=localhost --port=27017 --username=admin --password=secret \  --authenticationDatabase=admin --drop /home/manjiri/dump/d0919_744


5. Restore a single database

python dump_restore.py config.cfg --restore --db testdb1

Command run:

mongorestore --host=localhost --port=27017 --username=admin --password=secret \  --authenticationDatabase=admin --drop --nsInclude=testdb1.* /home/manjiri/dump/d0919_744


6. Restore specific collections from a database

python dump_restore.py config.cfg --restore --db testdb2:employees,projects


Mongo command run:

mongorestore --host=localhost --port=27017 --username=admin --password=secret \
  --authenticationDatabase=admin --drop \
  --nsInclude=testdb2.employees --nsInclude=testdb2.projects \
  /home/manjiri/dump/d0919_744


# How It Works Internally-
The script is structured into clear functions:

1. parse_db_args(db_args)

Converts CLI --db arguments into a mapping of { database_name: [collections] }.

Example:
["testdb1", "testdb2:employees,projects"]
→ { "testdb1": [], "testdb2": ["employees", "projects"] }

2. build_dump_cmds(...)

Constructs one or more mongodump commands depending on user input:

If --all, dumps all databases.
If --db dbname, dumps the whole database.
If --db dbname:coll1,coll2, dumps specific collections.

3. build_restore_cmd(...)

Constructs a single mongorestore command:

Adds --nsInclude filters for specific DBs/collections.
Always includes --drop to overwrite existing data.

Restores either:
All databases (--all), or
Only specified DBs/collections.

4. main()

Loads config file.
Parses command-line arguments.
Decides whether to run dump or restore.
Prints and executes the generated MongoDB commands using subprocess.run.

# Example flow:
Tracing the full flow: from --db arguments → Python mapping → actual mongodump commands that this script runs.

db_collections drives the commands in build_dump_cmds().

1. CLI Input

Example command:

python dump_restore.py config.cfg --dump --db testdb1 --db testdb2:employees,projects

2. Parsed Arguments → db_collections

Inside the script:

db_args = ["testdb1", "testdb2:employees,projects"]
db_collections = parse_db_args(db_args)


Result:

db_collections = {
    "testdb1": [],                        # means all collections
    "testdb2": ["employees", "projects"]  # only these collections
}

3. Passing into build_dump_cmds

build_dump_cmds looks like this (simplified):

for db, collections in db_collections.items():
    if collections:  # specific collections
        for coll in collections:
            cmd = [
                "mongodump",
                f"--db={db}",
                f"--collection={coll}",
                f"--out={dump_path}"
            ]
    else:  # no collections -> dump full DB
        cmd = [
            "mongodump",
            f"--db={db}",
            f"--out={dump_path}"
        ]

4. Generated Commands

Using the mapping above, we will get 3 separate mongodump calls:

For testdb1 (all collections):

mongodump --host=localhost --port=27017 --username=admin --password=secret --authenticationDatabase=admin --out=/path/to/backup --db=testdb1


For testdb2:employees:

mongodump --host=localhost --port=27017 --username=admin --password=secret --authenticationDatabase=admin --out=/path/to/backup --db=testdb2 --collection=employees


For testdb2:projects:

mongodump --host=localhost --port=27017 --username=admin --password=secret --authenticationDatabase=admin --out=/path/to/backup --db=testdb2 --collection=projects

5. Execution

Finally, the script loops through the commands:

for dump_cmd in dump_cmds:
    print("Command:", " ".join(dump_cmd))
    subprocess.run(dump_cmd, check=True)


So:

It prints each command (for visibility/logging).w get 
Then executes it with subprocess.run.

Summary:

--db testdb1 → [] → dumps entire DB.

--db testdb2:employees,projects → ["employees","projects"] → dumps specific collections.

Script builds multiple mongodump commands, one per DB/collection, and runs them sequentially.
