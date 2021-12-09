# Recording rules

What are recording rules?
* for example if yiu have to apply complex computations on the fly from expression browser the output will be delayed , prom gives you a solution to this 
in the form `recording rules` 
* Recording rules allow you to precompute frequently needed or compute expensive expressions and save them as new set of time series in prometheus storage 
* support 2 types of rules , recording rules and alerting rules 


# Reload Prom configs on the fly

1. By sending sighup signal to the prometheus instance
   ex: `kill -HUP <process id of prom server>`

2. by sending a http post to the prometheus server 
   this basically sends a post request to the reload endpoint   
   `curl -X POST http://localhost:9090/-/reload`  (be sure you have enabled the lifecycle api or you will get an error)

   