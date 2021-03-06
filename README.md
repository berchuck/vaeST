Implementing the Two-Stage Variational Autoencoder for Spatiotemporal Data
================
Samuel I. Berchuck
2019-09-10

*This document provides supplementary code for the paper "Scalable Modeling of Spatiotemporal Data using the Variational Autoencoder: an Application in Glaucoma",* <https://arxiv.org/abs/1908.09195>.

*Also, see a clinical implementation of the variational autoencoder on bioRxiv by Berchuck et al (2019): "Estimating Rates of Progression and Predicting Future Visual Fields in Glaucoma Using a Deep Variational Autoencoder",* <https://www.biorxiv.org/content/10.1101/652487v1.abstract>.

Loading Packages
----------------

We begin by loading the `keras` package and assigning the backend functions.

``` r
library(keras)
K <- keras::backend()
```

We can't include our visual field data from the manuscript, so we will use the MNIST digits data that comes with `keras`. We begin by loading the data and then creating training, validation, and test datasets.

``` r
###Load MNIST data from keras
mnist <- dataset_mnist()

###Assign training/validation and test datasets + scale to (0, 1) 
train_val_raw <- mnist$train$x / 255
test_raw <- mnist$test$x / 255

###Separate training and validation
train_raw <- train_val_raw[1:50000, , ]
val_raw <- train_val_raw[50001:60000, , ]
```

Before defining the variational autoencoder, we resize the MNIST images so that they are `12 x 12` dimensional, just like our visual fields. The resizing is performed through a simple spline interpolation based on the method from <https://stackoverflow.com/questions/35786744/resizing-image-in-r>. The function to resize is as follows.

``` r
resize <- function(img, new_width = 12, new_height = 12) {
  new_img <- apply(img, 2, function(y) {return(spline(y, n = new_height)$y)})
  new_img <- t(apply(new_img, 1, function(y) {return(spline(y, n = new_width)$y)}))
  new_img[new_img < 0] <- 0
  return(new_img)
}
```

Now, we apply the function to the MNIST digits.

``` r
###Data declarations
train_resized <- array(dim = c(dim(train_raw)[1], 12, 12))
val_resized <- array(dim = c(dim(val_raw)[1], 12, 12))
test_resized <- array(dim = c(dim(test_raw)[1], 12, 12))

###Resize the images
for (s in 1:dim(train_raw)[1]) train_resized[s, , ] <- resize(train_raw[s, , ])
for (s in 1:dim(val_raw)[1]) val_resized[s, , ] <- resize(val_raw[s, , ])
for (s in 1:dim(test_raw)[1]) test_resized[s, , ] <- resize(test_raw[s, , ])

###Resizing changed scale, so rescale
train_resized_scale <- train_resized / max(train_resized)
val_resized_scale <- val_resized / max(val_resized)
test_resized_scale <- test_resized / max(test_resized)
```

Finally, make the data have four dimensions as `keras` requires (sample size, height, width, depth).

``` r
###Create data objects for keras model
x_train <- array(dim = c(dim(train_resized_scale)[1], 12, 12, 1))
x_val <- array(dim = c(dim(val_resized_scale)[1], 12, 12, 1))
x_test <- array(dim = c(dim(test_resized_scale)[1], 12, 12, 1))

###Fill keras data objects
x_train[ , , , 1] <- train_resized_scale
x_val[ , , , 1] <- val_resized_scale
x_test[ , , , 1] <- test_resized_scale
```

We obviously do not suggest down-grading the quality of an image, but only are doing this so that the model architecture can be the same as in our analysis for the `12 x 12` dimensional visual fields. We can verify that the resizing was done properly, by visualizing an example image before and after resizing. Clearly, the new images are of poor quality, but they serve the purpose of our example.

``` r
par(mfcol = c(1, 2))
plot(as.raster(train_raw[1, , ]))
plot(as.raster(train_resized_scale[1, , ]))
```

<img src="README_files/figure-markdown_github/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

Model Architecture
------------------

We begin by setting the model parameters that will be used throughout the model architecture. In `keras` the arguments must be integers. In this code, we use a latent space with two dimension, which was chosen for visualization purposes only. We also set the seed using the `keras` function `use_session_with_seed`.

``` r
###Model parameters
batch_size <- 100L
original_dim <- 12L
latent_dim <- 2L
intermediate_dim <- 32L
final_dim <- 64L
epochs <- 50L
strides <- 2L
ks <- 3L

###Set keras seed
keras::use_session_with_seed(54)
```

    ## Set session seed to 54 (disabled GPU, CPU parallelism)

We then start model building be specifying the layers of the encoder. The encoder of the variational autoencoder is comprised of two 2D convolutional layers, a reshaping layer and a fully-connected dense layer. The result of the encoder is transforming the original `12 x 12` dimensional images into an `8` dimensional latent vector.

``` r
x <- layer_input(shape = c(original_dim, original_dim, 1))
C1 <- layer_conv_2d(x, filters = intermediate_dim, kernel_size = ks, strides = strides, activation = "relu")
C2 <- layer_conv_2d(C1, filters = final_dim, kernel_size = ks, strides = strides, activation = "relu")
R1 <- layer_flatten(C2)
F1 <- layer_dense(R1, units = latent_dim)
z <- F1 # define the latent vector space

###Define the keras sequential model
encoder <- keras_model(x, z)
```

Unlike the vanilla variational autoencoder we do not require a transformation step to sample the latent space. As such, we proceed to defining the decoder. The decoder begins with a fully-connected dense layer, followed by a reshaping layer, and then a sequence of deconvolutions, which up-sample until the original `12 x 12` dimension is reached. All layers use a `3 x 3` kernel size and a stride of `2`, and the activation is a rectified non-linear unit transformation, except for F1 and D3, which used the identity and sigmoid activations, respectively.

``` r
###Instantiate keras layer for decoder to be used later
F2 <- layer_dense(units = ks * ks * intermediate_dim, activation = "relu")
R2 <- layer_reshape(target_shape = c(ks, ks, intermediate_dim))
D1 <- layer_conv_2d_transpose(filters = final_dim, kernel_size = ks, strides = strides, padding = "same", activation = "relu")
D2 <- layer_conv_2d_transpose(filters = intermediate_dim, kernel_size = ks, strides = strides, padding = "same", activation = "relu")
D3 <- layer_conv_2d_transpose(filters = 1, kernel_size = ks, strides = 1, padding = "same", activation = "sigmoid")

###Define sequential model for decoder
z_1 <- layer_input(shape = latent_dim)
f2_1 <- F2(z_1)
r2_1 <- R2(f2_1)
d1_1 <- D1(r2_1)
d2_1 <- D2(d1_1)
x_pred_decoder <- D3(d2_1)
decoder <- keras_model(z_1, x_pred_decoder)
```

Finally, we define a sequential model for the entire variational autoencoder.

``` r
###Define sequential model for full variational autoencoder
f2 <- F2(z)
r2 <- R2(f2)
d1 <- D1(r2)
d2 <- D2(d1)
x_pred <- D3(d2)

###Full variational autoencoder
vae <- keras_model(x, x_pred)
print(vae)
```

    ## Model
    ## ___________________________________________________________________________
    ## Layer (type)                     Output Shape                  Param #     
    ## ===========================================================================
    ## input_1 (InputLayer)             (None, 12, 12, 1)             0           
    ## ___________________________________________________________________________
    ## conv2d_1 (Conv2D)                (None, 5, 5, 32)              320         
    ## ___________________________________________________________________________
    ## conv2d_2 (Conv2D)                (None, 2, 2, 64)              18496       
    ## ___________________________________________________________________________
    ## flatten_1 (Flatten)              (None, 256)                   0           
    ## ___________________________________________________________________________
    ## dense_1 (Dense)                  (None, 2)                     514         
    ## ___________________________________________________________________________
    ## dense_2 (Dense)                  (None, 288)                   864         
    ## ___________________________________________________________________________
    ## reshape_1 (Reshape)              (None, 3, 3, 32)              0           
    ## ___________________________________________________________________________
    ## conv2d_transpose_1 (Conv2DTransp (None, 6, 6, 64)              18496       
    ## ___________________________________________________________________________
    ## conv2d_transpose_2 (Conv2DTransp (None, 12, 12, 32)            18464       
    ## ___________________________________________________________________________
    ## conv2d_transpose_3 (Conv2DTransp (None, 12, 12, 1)             289         
    ## ===========================================================================
    ## Total params: 57,443
    ## Trainable params: 57,443
    ## Non-trainable params: 0
    ## ___________________________________________________________________________

Define the Loss
---------------

The variational autoencoder in our paper uses the maximum mean discrepancy (MMD) in place of the Kulback-Liebler divergance, so we define custom loss functions using the `keras` backend functions. We begin by defining the MMD, which is used as the regularization loss.

``` r
###Kernel function used in MMD calculation
compute_kernel <- function(x, y) {
  x_size <- k_shape(x)[1]
  y_size <- k_shape(y)[1]
  dim <- k_shape(x)[2]
  tiled_x <- k_cast(K$tile(k_reshape(x, k_stack(list(x_size, 1L, dim))), k_stack(list(1L, y_size, 1L))), "float32")
  tiled_y <- k_cast(K$tile(k_reshape(y, k_stack(list(1L, y_size, dim))), k_stack(list(x_size, 1L, 1L))), "float32")
  k_exp(-k_mean(k_square(tiled_x - tiled_y), axis = 3) / k_cast(dim, "float32"))
}

###Function to compute MMD
compute_mmd <- function(x, y, sigma_sqr = 1) {
  x_kernel <- compute_kernel(x, x)
  y_kernel <- compute_kernel(y, y)
  xy_kernel <- compute_kernel(x, y)
  k_mean(x_kernel) + k_mean(y_kernel) - k_cast(2L, "float32") * k_mean(xy_kernel)
}
```

Finally, we define our entire loss function, which combines the regularization loss using MMD and the reconstruction loss, which is a simple squared distance. We also, define a custom metric to be followed during model fit.

``` r
###Complete loss function
vae_loss <- function(x, x_pred) {
  dimz <- k_shape(z)[1]
  true_samples <- k_random_normal(shape = c(dimz, latent_dim), dtype = "float32")
  loss_reg <- compute_mmd(true_samples, z)
  loss_rec <- k_mean(k_square(k_cast(x, "float32") - k_cast(x_pred, "float32")))
  loss_reg + loss_rec
}

###Reconstruction loss to be plotted during fitting
metric_rec <- custom_metric("rec", function(x, x_pred) {
  k_mean(k_square(k_cast(x, "float32") - k_cast(x_pred, "float32")))
})
```

Fitting the Model
-----------------

The model was trained using the Adam optimizer, an extension of stochastic gradient descent, using 50 epochs and a batch size of 100; we used a learning rate of 1e-4.

``` r
###Optimization routine
compile(vae, optimizer = optimizer_adam(1e-4), loss = vae_loss, metrics = c(metric_rec))
```

Now we present code for training the model, where the training epoch with the minimal validation loss was chosen as optimal. The training is only performed if `RERUN` is set to `TRUE`, otherwise previously trained weights are loaded.

``` r
###Model training
filepath <- "./Raw/vae.h5"
RERUN <- FALSE # change to FALSE to use saved model weights
if (RERUN) {
  cp_callback <- callback_model_checkpoint(filepath = filepath, save_weights_only = FALSE, save_best_only = TRUE)
  vae.fit <- fit(vae,
                 x_train, x_train,
                 shuffle = TRUE,
                 epochs = epochs,
                 batch_size = batch_size,
                 validation_data = list(x_val, x_val),
                 metrics = c("loss", "rec"),
                 callbacks = list(cp_callback),
                 view_metrics = TRUE)
  save(vae.fit, file = "./Raw/fit.RData")
  save_model_hdf5(encoder, filepath = "./Raw/encoder.h5")
  save_model_hdf5(decoder, filepath = "./Raw/decoder.h5")
} else {
  load("./Raw/fit.RData")
  vae <- load_model_hdf5(filepath = "./Raw/vae.h5", custom_objects = c(vae_loss = vae_loss, rec = metric_rec))
  encoder <- load_model_hdf5(filepath = "./Raw/encoder.h5", custom_objects = c(vae_loss = vae_loss, rec = metric_rec))
  decoder <- load_model_hdf5(filepath = "./Raw/decoder.h5", custom_objects = c(vae_loss = vae_loss, rec = metric_rec))
}
```

Visualizing the overall and reconstruction loss is straightforward.

``` r
plot(vae.fit)
```

<img src="README_files/figure-markdown_github/unnamed-chunk-16-1.png" style="display: block; margin: auto;" />

Exploring the trained model
---------------------------

Once training has completed, we can use the `encoder`, `decoder`, and `vae` using the `predict` function. Here, we encode all of the images in the test dataset and then generate decoded images for each set of latent variables. We can see that the trained model is working adequetly, as a reconstructed image is shown to be similar to its original digit.

``` r
###Obtain latent space
latent <- predict(encoder, x_test)

###Generate an image
gen <- predict(decoder, latent)

###Plot an image and its decoded image 
par(mfcol = c(1, 2))
plot(as.raster(gen[3, , , ]))
plot(as.raster(x_test[3, , , ]))
```

<img src="README_files/figure-markdown_github/unnamed-chunk-17-1.png" style="display: block; margin: auto;" />

Visualizing the latent space
----------------------------

Begin by loading necessary packages.

``` r
library(reshape2)
library(ggplot2)
library(scales)
```

Since we are using two latent dimensions, we can visualize the latent space across digit type. This does not perform particularly well because we have down-sized the digits, however a pattern still forms.

``` r
###Extract test labels from MNIST data
Label <- as.factor(mnist$test$y)
test_out <- data.frame(latent, Label)

###Plot latent space
p <- ggplot(mapping = aes(latent[, 1], latent[, 2], color = Label))
p <- p + geom_point()
p <- p + xlab("Latent Dimension: 1") + ylab("Latent Dimension: 2")
print(p)
```

<img src="README_files/figure-markdown_github/unnamed-chunk-19-1.png" style="display: block; margin: auto;" />

Now, we can visualize the latent space using generated visual fields to see the data manifold in two-dimensions. Again, the visualization is not terribly clear due to the down-sizing of the MNIST digits for the training process.

``` r
###Create a dataset for visualization that is 20 x 20 digits.
n <- 20
img_size <- 12
Min <- apply(latent, 2, min)
Max <- apply(latent, 2, max)
grid_x <- seq(Min[1], Max[1], length.out = n)
grid_y <- seq(Min[2], Max[2], length.out = n)
rows <- NULL
for (i in 1:length(grid_x)) {
  column <- NULL
  for (j in 1:length(grid_y)) {
    z_sample <- matrix(c(grid_x[i], grid_y[j]), ncol = 2)
    pred <- predict(decoder, z_sample)[ , , , 1]
    column <- rbind(column, pred[dim(pred)[1]:1, ])
  }
  rows <- cbind(rows, column)
}

###Visualize two dimensional data manifold
data <- melt(t(rows))
p <- ggplot(data, aes(Var1, Var2, fill = value))
p <- p + geom_tile()
p <- p + scale_fill_gradientn(colours = c("black", "white"), limits = c(0, 1))
p <- p + coord_fixed()
p <- p + theme(panel.background = element_blank(),
               panel.border = element_blank(),
               panel.grid.major = element_blank(),
               panel.grid.minor = element_blank(),
               plot.background = element_blank(),
               axis.text.x = element_blank(),
               axis.text.y = element_blank(),
               axis.ticks = element_blank())
p <- p + labs(fill = "") + ylab("Latent Dimension: 2") + xlab("Latent Dimension: 1")
print(p)
```

<img src="README_files/figure-markdown_github/unnamed-chunk-20-1.png" style="display: block; margin: auto;" />

A Two-Stage Technique for Longitudinal Data
-------------------------------------------

In order to implement the two-stage method for spatiotemporal data, we only need to use the trained `decoder` function. The MNIST data does not inherently have a longitudinal component, however we can manufacture one. Assume that for a subject we observe a digit each year, with the first four being as follows.

``` r
###Define data for longitudinal example
Time <- 1:4
latent_pred <- matrix(c(0, -0.5, -1, -1.5, -2.5, -2, -1.5, -1), ncol = 2)

###Plot decoded images that we assume are truly observed annually
img_pred <- predict(decoder, latent_pred)
par(mfcol = c(1, 4))
for (y in 1:4) plot(as.raster(img_pred[y, , , 1]))
```

<img src="README_files/figure-markdown_github/unnamed-chunk-21-1.png" style="display: block; margin: auto;" />

Now, we can apply the simple two-stage technique by applying regression to each dimension of the latent space. This allows us to obtain a predicted image at year five.

``` r
###Fit linear regression independently to each latent dimension
mods <- apply(latent_pred, 2, function(x) lm(x ~ Time))

###Predict each dimension at the fifth year
latent_future <- matrix(unlist(lapply(mods, function(x) predict(x, newdata = data.frame(Time = 5)))), nrow = 1)

###Obtain the predicted image and plot
img_future <- predict(decoder, latent_future)[1 , , , 1]
plot(as.raster(img_future))
```

<img src="README_files/figure-markdown_github/unnamed-chunk-22-1.png" style="display: block; margin: auto;" />

While the MNIST digits do not have an inherent longitudinal nature this demonstrates the two-stage technique. When applied to spatiotemporal data, like visual fields, movement through the manifold corresponds to glaucoma disease progression, and is therefore clinically valuable.

Parts of this code were adapted from the following fabulous resources about variational autoencoders in `R`: <https://keras.rstudio.com/articles/examples/variational_autoencoder.html> and <https://blogs.rstudio.com/tensorflow/posts/2018-10-22-mmd-vae/>.
