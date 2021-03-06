
[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **MTS_afns_uk** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml

Name of QuantLet : MTS_afns_uk

Published in : MTS

Description : 'Shows the estimation results derived from the AFNS model in a multi-maturity term
structure for U.K.. Graphic showing the filtered and predicted state variables.'

Keywords : bond, graphical representation, plot, time-series, visualization

See also : MTS_afns_de, MTS_afns_sw, MTS_afns_fr, MTS_afns_it

Author : Shi Chen

Datafile : ukspot_nom.csv, ukspot_real.csv

Example : 'The estimated four latent factors of state variable for U.K.. The predicted are
presented as line type and the filtered are dashed.'

```

![Picture1](MTS_afns_uk.png)


### R Code:
```r
# ------------------------------------------------------------------------------
# Project:     MTS - Modeling of Term Structure for Inflation Estimation
# ------------------------------------------------------------------------------
# Quantlet:    MTS_afns_uk
# ------------------------------------------------------------------------------
# Description: The estimation results for U.K. derived from the AFNS model in 
#              multi-maturity term structrue. Graphic showing the filtered and 
#              predicted state variables.
# ------------------------------------------------------------------------------
# Keywords:    Kalman filter, optimization, MLE, maximum likelihood, bond, plot,
#              filter, estimation, extrapolation, dynamics, term structure,
#              interest-rate
# ------------------------------------------------------------------------------
# See also:
# ------------------------------------------------------------------------------
# Author:      Shi Chen
# ------------------------------------------------------------------------------

## clear history
rm(list = ls(all = TRUE))
graphics.off()

## install and load packages
libraries = c("zoo", "FKF", "expm", "Matrix")
lapply(libraries, function(x) if (!(x %in% installed.packages())) {
  install.packages(x)
})
lapply(libraries, library, quietly = TRUE, character.only = TRUE)

## read data of U.K.
ukdata1 = read.csv("ukspot_nom.csv", header = F, sep = ";")  #U.K. nominal bonds
ukdate = as.character(ukdata1[, 1])
st = which(ukdate == "30 Jun 06")
et = which(ukdate == "31 Dez 14")
ukdata11 = ukdata1[(st:et), 3:51]

ukdata2 = read.csv("ukspot_real.csv", header = F, sep = ";")  #U.K. inflation-indexed bonds
ukdate = as.character(ukdata2[, 1])
st = which(ukdate == "30 Jun 06")
et = which(ukdate == "31 Dez 14")
ukdata22 = ukdata2[(st:et), 2:47]
uknom = cbind(ukdata11[, 5], ukdata11[, 7], ukdata11[, 9])
ukinf = cbind(ukdata22[, 2], ukdata22[, 4], ukdata22[, 6])
ukmat = c(3, 4, 5)

ukjoi = cbind(uknom[, 1], ukinf[, 1])
for (i in 2:length(ukmat)) {
  ukjoi = cbind(ukjoi, uknom[, i], ukinf[, i])
}

y51 = t(ukjoi)

yieldadj_joint = function(sigma11, sigma12 = 0, sigma13 = 0, sigma21 = 0, 
                          sigma22, sigma23 = 0, sigma31 = 0, sigma32 = 0, sigma33, sigma44, lambda, 
                          time = maturity) {
  Atilde = sigma11^2 + sigma12^2 + sigma13^2 + sigma44^2
  Btilde = sigma21^2 + sigma22^2 + sigma23^2
  Ctilde = sigma31^2 + sigma32^2 + sigma33^2
  Dtilde = sigma11 * sigma21 + sigma12 * sigma22 + sigma13 * sigma23
  Etilde = sigma11 * sigma31 + sigma12 * sigma32 + sigma13 * sigma33
  Ftilde = sigma21 * sigma31 + sigma22 * sigma32 + sigma23 * sigma33
  
  adj1 = Atilde * time^2/6
  adj2 = Btilde * (1/(2 * lambda^2) - (1 - exp(-lambda * time))/(lambda^3 * 
                                                                   time) + (1 - exp(-2 * time * lambda))/(3 * lambda^3 * time))
  adj3 = Ctilde * (1/(2 * lambda^2) + exp(-lambda * time)/(lambda^2) - 
                     time * exp(-2 * lambda * time)/(4 * lambda) - 3 * exp(-2 * lambda * 
                                                                             time)/(4 * lambda^2) - 2 * (1 - exp(-lambda * time))/(lambda^3 * 
                                                                                                                                     time) + 5 * (1 - exp(-2 * lambda * time))/(8 * lambda^3 * time))
  adj4 = Dtilde * (time/(2 * lambda) + exp(-lambda * time)/(lambda^2) - 
                     (1 - exp(-lambda * time))/(lambda^3 * time))
  adj5 = Etilde * (3 * exp(-lambda * time)/(lambda^2) + time/(2 * lambda) + 
                     time * exp(-lambda * time)/lambda - 3 * (1 - exp(-lambda * time))/(lambda^3 * 
                                                                                          time))
  adj6 = Ftilde * (1/(lambda^2) + exp(-lambda * time)/(lambda^2) - exp(-2 * 
                                                                         lambda * time)/(2 * lambda^2) - 3 * (1 - exp(-lambda * time))/(lambda^3 * 
                                                                                                                                          time) + 3 * (1 - exp(-2 * lambda * time))/(4 * lambda^3 * time))
  
  return(adj1 + adj2 + adj3 + adj4 + adj5 + adj6)
}

Meloading_joint = function(lambda, alphaS, alphaC, time = maturity) {
  row1 = c(1, (1 - exp(-lambda * time))/(lambda * time), (1 - exp(-lambda * 
                                                                    time))/(lambda * time) - exp(-lambda * time), 0)
  row2 = c(0, alphaS * ((1 - exp(-lambda * time))/(lambda * time)), alphaC * 
             ((1 - exp(-lambda * time))/(lambda * time) - exp(-lambda * time)), 
           1)
  MatrixB = rbind(row1, row2)
  return(MatrixB)
}

afnsss = function(t1, t2, t3, t4, t5, t6, t7, t8, t9, t10, t11, t12, t13, 
                  t14, t15, t16, s1, s2, s3, s4, g1, g2, l1, h1, h2, h3, h4) {
  Tt = matrix(c(t1, t2, t3, t4, t5, t6, t7, t8, t9, t10, t11, t12, t13, 
                t14, t15, t16), nr = 4)
  Zt = rbind(Meloading_joint(lambda = l1, alphaS = g1, alphaC = g2, time = ukmat[1]), 
             Meloading_joint(lambda = l1, alphaS = g1, alphaC = g2, time = ukmat[2]), 
             Meloading_joint(lambda = l1, alphaS = g1, alphaC = g2, time = ukmat[3]))
  ct = matrix(rep(c(yieldadj_joint(sigma11 = h1, sigma22 = h2, sigma33 = h3, 
                                   sigma44 = h4, lambda = l1, time = ukmat[1]), yieldadj_joint(sigma11 = h1, 
                                                                                               sigma22 = h2, sigma33 = h3, sigma44 = h4, lambda = l1, time = ukmat[2]), 
                    yieldadj_joint(sigma11 = h1, sigma22 = h2, sigma33 = h3, sigma44 = h4, 
                                   lambda = l1, time = ukmat[1])), each = 2), nr = 6, nc = 1)
  dt = matrix(c(1 - t1, t2, t3, t4, t5, 1 - t6, t7, t8, t9, t10, 1 - 
                  t11, t12, t13, t14, t15, 1 - t16), nr = 4) %*% matrix(c(s1, s2, 
                                                                          s3, s4), nr = 4)
  GGt = matrix(0.1 * diag(6), nr = 6, nc = 6)
  H = diag(c(h1^2, h2^2, h3^2, h4^2), nr = 4)
  HHt = Tt %*% H %*% t(Tt)
  a0 = c(s1, s2, s3, s4)
  P0 = HHt * 10
  return(list(a0 = a0, P0 = P0, ct = ct, dt = dt, Zt = Zt, Tt = Tt, GGt = GGt, 
              HHt = HHt))
}

## The objective function passed to 'optim'
objective = function(theta, yt) {
  sp = afnsss(theta["t1"], theta["t2"], theta["t3"], theta["t4"], theta["t5"], 
              theta["t6"], theta["t7"], theta["t8"], theta["t9"], theta["t10"], 
              theta["t11"], theta["t12"], theta["t13"], theta["t14"], theta["t15"], 
              theta["t16"], theta["s1"], theta["s2"], theta["s3"], theta["s4"], 
              theta["g1"], theta["g2"], theta["l1"], theta["h1"], theta["h2"], 
              theta["h3"], theta["h4"])
  ans = fkf(a0 = sp$a0, P0 = sp$P0, dt = sp$dt, ct = sp$ct, Tt = sp$Tt, 
            Zt = sp$Zt, HHt = sp$HHt, GGt = sp$GGt, yt = yt)
  return(-ans$logLik)
}

theta <- c(t = c(0.8, 0, 0, 0, 0, 0.8, 0, 0, 0, 0, 0.8, 0, 0, 0, 0, 0.8), 
           s = c(0.2, 0.2, 0.2, 0.2), g = c(0.6, 0.6), l1 = c(0.7), h = c(0.2, 
                                                                          0.2, 0.2, 0.2))

fit = optim(theta, objective, yt = y51, hessian = TRUE)
sp = afnsss(fit$par["t1"], fit$par["t2"], fit$par["t3"], fit$par["t4"], 
            fit$par["t5"], fit$par["t6"], fit$par["t7"], fit$par["t8"], fit$par["t9"], 
            fit$par["t10"], fit$par["t11"], fit$par["t12"], fit$par["t13"], fit$par["t14"], 
            fit$par["t15"], fit$par["t16"], fit$par["s1"], fit$par["s2"], fit$par["s3"], 
            fit$par["s4"], fit$par["g1"], fit$par["g2"], fit$par["l1"], fit$par["h1"], 
            fit$par["h2"], fit$par["h3"], fit$par["h4"])
ans = fkf(a0 = sp$a0, P0 = sp$P0, dt = sp$dt, ct = sp$ct, Tt = sp$Tt, Zt = sp$Zt, 
          HHt = sp$HHt, GGt = sp$GGt, yt = y51)

res = matrix(rowMeans(ans$vt[, 2:103]), nr = 6)
joiuk0915ans = ans
joiuk0915fit = fit
save(joiuk0915ans, file = "joiuk0915ans.RData")
save(joiuk0915fit, file = "joiuk0915fit.RData")

## The plots of filtered and predicted state variables 
## Another approach: plot.fkf(ans, CI=NA)
plot(ans$at[1, -1], type = "l", col = "red", ylab = "State variables", 
     xlab = "", ylim = c(-6, 6), lwd = 2)
lines(ans$att[1, -1], lty = 2, col = "red", lwd = 2)
lines(ans$at[2, -1], lty = 1, col = "purple", lwd = 2)
lines(ans$att[2, -1], lty = 2, col = "purple", lwd = 2)
lines(ans$at[3, -1], lty = 1, col = "grey3", lwd = 2)
lines(ans$att[3, -1], lty = 2, col = "grey3", lwd = 2)
lines(ans$at[4, -1], lty = 1, col = "blue", lwd = 2)
lines(ans$att[4, -1], lty = 2, col = "blue", lwd = 2)


```
