{spawn, exec} = require 'child_process'
fs = require 'fs'
path = require 'path'
log = console.log
app = "app/app.coffee"

cachedPath = 'static/data/cached'


option '-e', '--environment [ENVIRONMENT_NAME]', 'set the environment for `restart`'

mkdir = (dir) -> if !path.existsSync(dir) then fs.mkdirSync(dir, parseInt('0755', 8))

task 'build', ->
  mkdir "static"
  mkdir "static/coffee"
  mkdir "static/coffee/charts"

  run 'coffee -o static/coffee -c app/frontend/*.coffee app/frontend/*/*.coffee'
  run 'coffee -o static/coffee/charts -c app/charts/*.coffee'


# See: http://stackoverflow.com/questions/7259232/how-to-deploy-node-js-in-cloud-for-high-availability


task 'restart', 'Build, kill existing app processes and run it again', (options) ->
  options.environment or= 'production'
  invoke 'build'
  run 'PATH=/usr/bin:/usr/local/bin  && kill -9 `pgrep -f "coffee '+app+'"`'

  run "NODE_ENV=#{options.environment} coffee "+app


forever = (action, options) ->
  invoke 'build'
  options.environment or= 'production'
  #if !path.existsSync('logs') then fs.mkdir('logs', parseInt('0755', 8))
  run "NODE_ENV=#{options.environment} " +
      "node_modules/forever/bin/forever #{action} -c coffee " +
      " --sourceDir ./" +
      " -l aiddata.log "+
      #" -o logs/aiddata.out "+
      #" -e logs/aiddata.err "+
      " -a" +   # append logs
      " " + app


task 'restart-forever', (options) -> forever 'restart', options
task 'start-forever', (options) -> forever 'start', options
task 'stop-forever', (options) -> run "node_modules/forever/bin/forever stop -c coffee " + app
task 'list-forever', (options) -> run "node_modules/forever/bin/forever list"


task 'mongo-imports', ->  require('./data/mongo-imports').run()


task 'refresh-views', ->
  pg = require('./pg-sql').include()

  views = [
      "donor_recipient_year_purpose_ma",
      "totals_for_donor_ma",
      "totals_for_recipient_ma",
      "totals_for_recipient_to_donor_ma" ]

  console.log "Refreshing views #{views}"

  for view in views
    pg.sql "select '#{view}' as view, refresh_matview('#{view}')",
      (err, data) =>
        v = data.rows[0].view
        unless err?
          console.log "Success refreshing #{v}"
        else
          console.log "Refreshing #{v} failed", err
    


task 'refresh-cached', ->
  mkdir cachedPath

  files = [
    "aiddata-donor-totals.csv",  
    "aiddata-recipient-totals.csv", 
    "flows.json",
    "aiddata-nodes.csv", 
    "aiddata-totals-d-r-y.csv"]
  for file in files
    console.log "Refreshing file #{file}"
    run  "wget http://localhost:3000/#{file} -O #{cachedPath}/#{file}"




run = (args...) ->
  for a in args
    switch typeof a
      when 'string' then command = a
      when 'object'
        if a instanceof Array then params = a
        else options = a
      when 'function' then callback = a
  
  command += ' ' + params.join ' ' if params?
  cmd = spawn '/bin/sh', ['-c', command], options
  cmd.stdout.on 'data', (data) -> process.stdout.write data
  cmd.stderr.on 'data', (data) -> process.stderr.write data
  process.on 'SIGHUP', -> cmd.kill()
  cmd.on 'exit', (code) -> callback() if callback? and code is 0
