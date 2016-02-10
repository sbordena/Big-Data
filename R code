### light beer solutions

install.packages("distrom")

library(distrom)
source("naref.R")
source("fdr.R")
source("roc.R")
source("deviance.R")

lb <- read.csv("LightBeer.csv")

# break out the variables of main interest
n <- nrow(lb)
logspend <- log(lb$beer_spend)
logprice <- log(lb$price_floz)
logvol <- log(lb$beer_floz)
brand <- lb$beer_brand
container <- lb$container_descr
promo <- lb$promo
demog <- lb[,-(1:9)]
purch <- lb[,1:3]

# relevel some things
demog$market <- relevel(demog$market, "CHICAGO")
demog$income <- factor(demog$income, levels=rev(levels(demog$income)))

########
## Q1 ##
########

### 1.1 
# MLE linear model fit
mle <- glm( logprice ~ logvol*promo*brand + container + ., data=demog )

summary(mle)

### 1.2
residuals <- logprice - predict(mle,demog,type="response")

plot(logvol, residuals)

logvol2<-logvol^2

mle_improved <- glm( logprice ~ logvol*promo*brand + container + logvol2 + ., data=demog )
summary(mle_improved)

residuals_improved <- logprice - predict(mle_improved,demog,type="response")
plot(logvol, residuals_improved)

### 1.3

## grab p-values
pvals <- summary(mle)$coef[-1,4] #-1 to drop the intercept
## plot them: it looks like we have some signal here
hist(pvals, xlab="p-value", main="", col="lightblue")

q <- 0.01
alpha <- fdr_cut(pvals, q) ## cutoff

q2 <- 0.005
alpha2 <- fdr_cut(pvals, q2) ## cutoff


### 1.5
# bootstrapping
B <- 20
volbeta <- rep(NA,B)
set.seed(41201)
for(b in 1:B){
    print(b)
    n <- nrow(demog)
    ib <- sample(1:n,n,replace=TRUE)
    mleb <- glm( logprice ~ logvol*promo*brand + container + ., data=demog, subset=ib )
    volbeta[b] <- coef(mleb)["logvol"]
}
hist(volbeta)

median(volbeta)
sort(volbeta)

confint(mle,"logvol",level=0.8)

########
## Q2 ##
########

# build a bigger demographics design by interacting with market
xdemog <- sparse.model.matrix( 
    ~ market*(buyertype+income+childrenUnder6+children6to17+
        employment+degree+occupation+ethnic+microwave+
        dishwasher+tvcable+singlefamilyhome+npeople
        ), data=naref(demog))[,-1]
xdemog <- xdemog[,colSums(xdemog)>0] # drop n'er occurs

xbeer <- sparse.model.matrix( ~ ., data=data.frame(brand=naref(brand), promo) )[,-1]

### 2.1

univariate <- glm(logspend ~ logprice)
summary(univariate)

lasso <- gamlr(cBind(logprice,xdemog,xbeer),logspend)

coef(lasso)["logprice",]

### 2.3

treat <- gamlr(cBind(xdemog,xbeer),logprice,lambda.min.ratio=1e-4)
logprice_hat <- predict(treat, cBind(xdemog,xbeer), type="response") 

plot(logprice_hat,logprice,bty="n",pch=21,bg=8) 

### 2.4
causal <- gamlr(cBind(logprice,logprice_hat,xdemog,xbeer),logspend,free=2,lmr=1e-4)
coef(causal)["logprice",]

# 2.5
# now subset to a small group
sub = which(demog$market=="LOS ANGELES" & demog$income=="200k+")

lasso2 <- gamlr(cBind(logprice,xdemog,xbeer)[sub,],logspend[sub])
coef(lasso2)["logprice",]

causal2 <- gamlr(cBind(logprice,logprice_hat,xdemog,xbeer)[sub,],logspend[sub],free=2,lmr=1e-4)
coef(causal2)["logprice",]

log(causal2$lambda)[which.min(AICc(causal2))]

########
## Q3 ##
########

## build a much bigger matrix of price interactions
xprice <- sparse.model.matrix( ~ logprice*(market*income+brand+promo)-market*income-brand-promo, 
    data=naref(cbind(demog,brand,promo)))[,-1]
xprice <- xprice[,colSums(xprice!=0)>0] # remove the never occurs
x <- cBind(xprice,xbeer,xdemog) # bind with out other matrices

# 3.1
cv_logspend <- cv.gamlr(x, logspend, verb=TRUE,lambda.min.ratio=1e-4)

cv_logspend$seg.min
cv_logspend$seg.1se

# 3.2

## comparing AICc, BIC, and the CV error
lasso3 <- gamlr(x, logspend, lambda.min.ratio=1e-4)

n<-nrow(x)
plot(cv_logspend, bty="n")

lines(log(lasso3$lambda),exp(AICc(lasso)/n), col="green", lwd=2, lty=1)
lines(log(lasso3$lambda),exp(AIC(lasso)/n), col="red", lwd=2, lty=1)
lines(log(lasso3$lambda),exp(BIC(lasso)/n), col="maroon", lwd=2, lty=1)
legend("top", fill=c("blue", "red","green","maroon"),
       legend=c("CV","AIC","AICc","BIC"), bty="n")

abline(v=log(lasso3$lambda)[which.min(AICc(lasso3))], col="green", lty=4)
abline(v=log(lasso3$lambda)[which.min(AIC(lasso3))], col="red", lty=3)
abline(v=log(lasso3$lambda)[which.min(BIC(lasso3))], col="maroon", lty=3)

# 3.3

coef(cv_logspend)["logprice:marketLOS ANGELES:income200k+",]
coef(cv_logspend)["logprice",]

# 3.4
# the median consumer
xbar_median <- apply(xprice/logprice, 2, median)

Contains_logprice<-grep("logprice+",row.names(coef(cv_logspend)), value=TRUE) #All rows that have interaction with price

coef_logprice<-coef(cv_logspend)[Contains_logprice,]

sum(xbar_median*coef_logprice)

# 3.5
## bootstrap using a for loop.
## note we don't need cv.gamlr here, just gamlr
set.seed(41201)
B <- 20 
DF <- rep(0,B)
for(b in 1:B){
    index <- sample.int(n,replace=TRUE)
    gamlrb <- gamlr(x[index,], logspend[index], lambda.min.ratio=1e-4)
    DF[b] <- gamlrb$df[which.min(AICc(gamlrb))]
    print(b)
}
hist(DF)
abline(v=lasso3$df[which.min(AICc(lasso3))], col="green", lty=4)
legend("top", fill=c("green"),
       legend=c("full sample AICc"), bty="n")


########
## Q4 ##
########

# 4.1

ybud <- rep(0,n)
for (b in 1:n) {
    if (brand[b] =="BUD LIGHT") { ybud[b] <-1
    } 
}

lasso_bud <- gamlr(xdemog, ybud,lambda.min.ratio=1e-4, family="binomial")
plot(lasso_bud)

ybud_predict <- predict(lasso_bud,xdemog, type="response")
plot(factor(ybud), ybud_predict,
  ylab="prediction",
  xlab="BUD LIGHT")

# 4.2
D<-lasso_bud$deviance[which.min(AICc(lasso_bud))]
D0<-lasso_bud$deviance[1]
Rsquared <- 1 - D/D0

# 4.3
sensi<-mean( (ybud_predict>0.1)[ybud==1] )# sensitivity
speci<-mean( (ybud_predict<0.1)[ybud==0] )# specificity

roc(p=ybud_predict, y=ybud, main="ROC for our BUD LIGHT prediction")
points(y=sensi,x=1-speci, col="green" )
legend("right", fill=c("green"),
       legend=c("10% rule"), bty="n")

# 4.4
xdemog[1,]

mean( (ybud_predict)[xdemog[,"marketRURAL CALIFORNIA"]==1])
coef(lasso_bud)["marketRURAL CALIFORNIA",]

coef(lasso_bud)["marketRURAL CALIFORNIA",]

Contains_Cali_income<-grep("marketRURAL CALIFORNIA:income+",row.names(coef(lasso_bud)), value=TRUE) #All rows that have interaction with california
coef(lasso_bud)[Contains_Cali_income,]
#### 4.5
library(parallel)
cl <- makeCluster(
    min(detectCores(),5),
    type=ifelse(.Platform$OS.type=="unix","FORK","PSOCK"))
multifit <- dmr(cl, xdemog, brand, verb=TRUE, lmr=1e-3)
multifit_Cali_income<-grep("marketRURAL CALIFORNIA:income+",row.names(coef(multifit)), value=TRUE)
coef(multifit)[multifit_Cali_income,]
