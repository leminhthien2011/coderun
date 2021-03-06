# coderun


#The code is created on Aug 12, 12 p.m. to recover the parameters combo
theta1 = c(0,.55,.23,.03,1,.25) # c(alpha0,alpha, beta, delta, eta, gamma
alpha = theta1[2]
beta = theta1[3]
delta = theta1[4]
eta = theta1[5]
gamma = theta1[6]
#alpha, alpha0: Transmission Rate
#beta: recover rate
#delta: death
#eta: relative recover unreported
#gamma: identification
###

#############
fname = paste('Datagen_alpha',alpha,"_beta",beta,"_delta",delta,"_eta",eta,"_gamma",gamma,".txt",sep="") 
mydata = read.table(fname,header=T) 
####################################
####Target of ABC is recovering the combo based on mydata available#############

library(protoABC)
library(ggplot2)
library(dplyr)
library(tidyr)
library(cowplot)

####### 1. We first define the generative function of the process


##############################################################
##################################################################################
#Tranmission rate act as a function updated by time
tran_rate = function(x,theta){
  tranmiss = theta[1] + theta[2]
  names(tranmiss) =c("tranmissionrate")
  return(tranmiss)
}

###########################
#The model is updated based on tau leaping method with hazard functions as:
#h1(Xt) = tran_rate(x)*x["S"]*x["I"]/P #
#h2(Xt) = theta["gamma"]*x["I"]
#h3(Xt) = theta["beta]*x["A"]
#h4(Xt) = theta["delta"]*x["A"]
#h5(Xt) = theta["eta"]*theta["beta"]*x["I"]
##################Deefining harzard functions
#New infected rate, alphas,c(alpha0,alpha, beta, delta, eta, gamma) 
harzard1 = function(x,theta){
  h1 = tran_rate(x,theta)*x[1]*x[2]/sum(x)
  names(h1)=c("hazard1")
  return(h1)
}

#New confirmed rate, gamma, c(alpha0,alpha, beta, delta, eta, gamma)
harzard2 = function(x,theta){
  h2 = theta[6]*x[2]
  names(h2)=c("hazard2")
  return(h2)
}
#New confirmed recover, beta, c(alpha0,alpha, beta, delta, eta, gamma)
harzard3 = function(x,theta){
  h3 = theta[3]*x[3]
  names(h3)=c("hazard3")
  return(h3)
}
#New confirmed death, delta, c(alpha0,alpha, beta, delta, eta, gamma)
harzard4 = function(x,theta){
  h4 = theta[4]*x[3]
  names(h4)=c("hazard4")
  return(h4)
}
#New unconfirmed recover, eta*beta, c(alpha0,alpha, beta, delta, eta, gamma)
harzard5 = function(x,theta){
  h5 = theta[5]*theta[3]*x[2]
  names(h5)=c("hazard5")
  return(h5)
}
##############Initial Conditions##########
#We know 10^7 is the population size
S1 = 10^6-I1 - A1
I1 = 5
A1 = 3
x1 = c(S1,I1,A1,0,0,0)# State corresponding S,I,A,R,D,Ru

##################



nrep = nrow(mydata)




##We define a generating function when for a given theta
generating_func = function(theta){
  status_matrix = matrix(0,nrow = nrep,ncol=6)
  status_matrix1 =  status_matrix
  status_matrix[1,] = x1
for (i in 2:nrep){
  
  x = status_matrix[(i-1),]
  ##Generating Poisson values based on hazard functions
  #y1 = rpois(1, harzard1(x,theta))
  y1 =  harzard1(x,theta)
  #y2 =  rpois(1, harzard2(x,theta))
  y2 =  harzard2(x,theta)
  
  #y3 = rpois(1, harzard3(x,theta))
  y3 =  harzard3(x,theta)
  
  #y4 =  rpois(1, harzard4(x,theta))
  y4 =  harzard4(x,theta)
  
  
  #y5 = rpois(1, harzard5(x,theta))
  y5 =  harzard5(x,theta)
  
  
  
  if(y1>x[1]){y1=x[1]}
  if(y1-y2-y5+x[2]<0){y2=x[2]+y1-y5}
  if(y2-y3-y4+x[3]<0){y3=x[3]+y2-y4}
  
  y=c(y1, y2, y3, y4, y5)
  ##Transition matrix
  tran_mat = matrix(c(-1,1,0,0,0,0,0,-1,1,0,0,0,0,0,-1,1,0,0,0,0,-1,0,1,0,0,-1,0,0,0,1),nrow=6,ncol=5)
  ##Updating values
  val = tran_mat%*%y
  x = x+ t(val)
  status_matrix[i,] = x
}

return(status_matrix)
}
#####################


#2. We define a prior
prior <- function(n){
  data.frame(
    alpha0 = 0,
    alpha = runif(n, 0.3, 1),
    beta = .23,
    delta=.03,
    eta=1,
    gamma=.25
  )
}

###############
# We define a distance function
inp = mydata[,1] #used only active confirmed

distance <- function(theta, inp){
  sim <- generating_func(theta)
  sim_active = sim[,3]
  output <- sqrt(sum((sim_active -inp)^2))
  return(output)
}


############## Perform ABC for alpha#####
tmp = proc.time()
eps= 1000
post_project <- abc_start(
  prior,
  distance,inp,
  method = "rejection",
  control = list(epsilon = eps, n = 20)
)
hist(post_project[,2])

########Define Mode function
# Create the function.


theta_project = apply(post_project, 2, median)


#################
predict_project = generating_func(theta_project)

###########



########We now plot the updating process and try to understand how the process evolve for each country


t = seq(1,nrow(mydata))
active = mydata[,1]
active_project = predict_project[,3]


mydata[,2]
mydata[,3]
data = data.frame(t,active, active_project)

colnames(data) =  c("Time", "active", "active_project")


##########1. Plot susceptible cases
data_long = gather(data, Class, Cases, active:active_project, factor_key=TRUE)
plt1 = ggplot(data_long, aes(x=Time, y=Cases, shape=Class, color=Class)) +
  geom_point()+labs(title = "Classes overtime")
plt1
##################

max(active)
max(active_project)
hist(post_project[,2])
theta_project
sqrt(sum(active - active_project)^2)
proc.time()-tmp
