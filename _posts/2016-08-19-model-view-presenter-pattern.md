---
layout: post
title:  "Model-View-Presenter Pattern"
date:   2016-08-19 12:00:00 +0200
comments: true
---

This presentation of the Model-View-Presenter Pattern may be sometimes influenced by how I use it with Vaadin.

## Introduction

The goal of this pattern is to separate the concerns of the presentation layer into three roles:

* the **model** moves the data to display from the presenter to the view
* the **view** binds the model properties to the visual components and delegates user interactions to the presenter
* the **presenter** handles user interactions and drives the view in order to react to events

Separating concerns improve the **testability** of the system. The system would be fully testable if all specifications could be asserted on the presenter and on the model.

![MVP overview](http://i.imgur.com/uxBtOaT.png)

In order to deeper explain the pattern let's dive into some examples. These ones comes from a fictive blog application and are deliberately simple. The sources can be find [here](https://github.com/erichonorez/mvp-example).

## Display data

The first example is very simple. The ``PostView`` view just renders the post having the identifier specified in the URL of the page. E.g. ``http://my.blog.com/posts/0f4d110b-3137-4bfb-8e2d-64099c9c3401``.

So in order to achieve it the following sequence of operations is executed:

1. the application routes the HTTP request to the ``PostView`` view;
3. the view tells the presenter to initialise it;
4. the presenter fetch the post with the identifier from the business layer;
5. it then creates a ``PostViewModel`` model with the post entity fetched from the business layer. This presentation model holds all information rendered by the view or needed to render it;
6. finally the presenter tells the view to bind its components to the model;

![Request sequence](http://i.imgur.com/mcglOW3.png)

Here is the unit test defining the specification:

```java

    @Test
    public void itShouldDisplayPostById() {
        // Given a post with id "0f4d110b-3137-4bfb-8e2d-64099c9c3401" exists
        String aPostId = "0f4d110b-3137-4bfb-8e2d-64099c9c3401";

		 // ... PostRepository mocking and stubbing...

        // When the user goes to the post page
        PostPresenter presenter = new PostPresenterImpl(postRepository);
        PostView view = mock(PostView.class);
        presenter.initView(view, aPostId);

        // Then the post is displayed
        ArgumentCaptor<PostViewModel> argument
	        = ArgumentCaptor.forClass(PostViewModel.class);

        verify(view).bind(argument.capture());

        PostViewModel viewModel = argument.getValue();

        assertThat(viewModel.title())
            .isEqualTo(aTitle);

        assertThat(viewModel.content())
            .isEqualTo(aContent);
    }
```

The APIs of the presenter and the view:

```java
/**
 * Controls the display of a post.
 */
public interface PostPresenter {

    /**
     * Initialise the view for the specified post identifier
     *
     * @param view the calling view
     * @param aPostId the identifier of the post to diplay
     */
    void initView(PostView view, String aPostId);

}

```

When the view is ready to be rendered it calls the ``initView`` method of its presenter passing its reference and the query parameters. In that case the only query parameter is the post identifier.

```java
/**
 * Displays a post to the user
 */
public interface PostView {

    /**
     * Bind properties of the PostViewModel to visual components.
     */
    void bind(PostViewModel viewModel);

}
```

After having fetched data from a persistence storage the presenter builds a ``PostViewModel`` model and tells the view to bind visual components by calling the ``bind`` method.

Here is the implementation of the presenter:

```java
public class PostPresenterImpl implements PostPresenter {

    private PostRepository postRepository;

    public PostPresenterImpl(PostRepository postRepository) {
        this.postRepository = postRepository;
    }

    public void initView(PostView view, String aPostId) {
        JsonObject post = this.postRepository.findById(aPostId);

        view.bind(
            new PostViewModel(
                post.get("title").getAsString(),
                post.get("content").getAsString()));
    }

}
```

###  Handle request parameters

``PostPresenter#initView`` receive all necessary arguments to create a ``PostViewModel``. It must receive all information it needs in order to do its job.

If the information the view displays is internationalized, a locale must be passed to the presenter. If the view depends on the current user authorization the presenter must receive the current user in parameter.

Other common parameters may be:
 * The query parameters;
 * The pagination parameters (offset, limit).

Imagine that the posts are available on several languages and depend on the current user, the signature of the ``initView``method might be:

```java
public class PostListPresenter {
  /**
   * @param aView the calling view
   * @param aLocale the locale in which the posts must be displayed
   * @param aUser the logged user
   * @param offset the number of first posts to skip
   * @param limit the number of posts to return
   */
	public initView(
    PostListView aView,
    Locale aLocale,
    User aUser,
    int offset,
    int limit) {

		Model model = //fetch data from db and do some computations
		// the presenter tell the view to bind the model
		view.bind(model);
	}
}
```

In that case the model would contain the list of posts with pagination information:
* the total number of pages
* the current page

The **model** contains all necessary information to render the view without having logic in the view. The view only knows how to render things and not what to render.

## Tell the view to do
The presenter must tell the view what to do. The view should never take any decision to do something when an event occurs.

### Handling exceptions
It is for that reason that exceptions should never be handled by the view. The presenter should catch exceptions and tell the view what to do.

Let's imagine the case where the visitor tries to access a post that does not exists. In that scenario the application must:

1. notify the user the post he tried to access does not exist;
2. redirect the user to the list of posts.

Here are the unit tests asserting the specification:

```java

public static class WhenPostDoesNotExist {

        @Test
        public void itShouldNotifyTheUser()
	        throws BusinessObjectNotFoundException {
            // Given a post with id "0f4d110b-3137-4bfb-8e2d-64099c9c3401" does not exist
            String aPostId = "0f4d110b-3137-4bfb-8e2d-64099c9c3401";

            PostRepository postRepository = mock(PostRepository.class);
            when(postRepository.findById(aPostId))
	            .thenThrow(new BusinessObjectNotFoundException());

            // When the user goes to the post page
            PostPresenter presenter = new PostPresenterImpl(postRepository);
            PostView view = mock(PostView.class);
            presenter.initView(view, aPostId);

            verify(view).notifyError(anyString());
        }

        @Test
        public void itShouldRedirectTheUserToPostList() throws
	        BusinessObjectNotFoundException {
            // Given a post with id "0f4d110b-3137-4bfb-8e2d-64099c9c3401" does not exist
            String aPostId = "0f4d110b-3137-4bfb-8e2d-64099c9c3401";

            PostRepository postRepository = mock(PostRepository.class);
            when(postRepository.findById(aPostId))
	            .thenThrow(new BusinessObjectNotFoundException());

            // When the user goes to the post page
            PostPresenter presenter = new PostPresenterImpl(PostRepository);
            PostView view = mock(PostView.class);
            presenter.initView(view, aPostId);

            verify(view).navigateToPostList();
        }

    }
    
```

If we are looking at the business component that fetches the post by its id:

```java

/**
 * Re-creates Post from storage.
 */
public interface PostRepository {

    /**
     * Fetch a post form persistence storage or throw
     * BusinessObjectNotFoundException
     * if not found.
     */
    public JsonObject findById(String aPostId)
	    throws BusinessObjectNotFoundException;

}
```

So the goal is to avoid the exception to leak into the view and give the view to responsibility to handle it. So the ``initView`` method of the presenter:


```java

public class PostPresenterImpl implements PostPresenter {

	// ... code

	public void initView(PostView view, String aPostId) {
       JsonObject post;
       try {
           post = this.postRepository.findById(aPostId);

           view.bind(
               new PostViewModel(
                   post("title").getAsString(),
                   post("content").getAsString()));

       } catch (BusinessObjectNotFoundException e) {
           view.notifyError("The post does not exist");
           view.navigateToPostList();
       }
   }

   // ... code
}

```

## Internationalization

All data coming from the presenter that need to be translated when displayed must be returned to the view translated.

Data that does not come from the presenter are internationalized by the view. E.g. labels and all other static values.


## Handling forms

Once again the goal is to be able to test form validation by unit testing. So the validation must be on the presenter side.

Once the user what's to create a post or to update existing one here are the rules applied:

- The title of the post
	- cannot be null
	- length must be between 1 and 100
- The content of the
	- cannot be null
	- length must be between 1 and 2048

We are going to use the **Java Bean Validation API** in order to validate the form object.

```java

public class PostForm {

    @NotNull
    @Size(min = 1, max = 100)
    public String title;

    @NotNull
    @Size(min = 1, max = 2048)
    public String content;

}

```

Here are the unit tests asserting the specifications:

```java

public static class WhenFormIsNotValid {

        @Test
        public void itShouldDisplayErrorWhenTitleIsEmpty() {
            // When the title of the form has not been filled in
            PostForm form = new PostForm();

            // Then the error must be displayed to the user
            Validator validator = Validation
                .buildDefaultValidatorFactory().getValidator();

            PostEditionPresenter presenter
	            = new PostEditionPresenterImpl(validator, mock(PostRepository.class));
            PostEditionView view = mock(PostEditionView.class);

            presenter.save(view, form);

            Class<Set<ConstraintViolation<PostForm>>> argumentClass =
                (Class<Set<ConstraintViolation<PostForm>>>) (Class) Set.class;

            ArgumentCaptor<Set<ConstraintViolation<PostForm>>> argument =
                ArgumentCaptor.forClass(argumentClass);

			// Then the application should show errors to user
            verify(view).displayErrors(argument.capture());

            Set<ConstraintViolation<PostForm>> validations = argument.getValue();
            BWValidationAssert.fails(validations, "title");
        }

    }


    public static class WhenFormIsValid {

        @Test
        public void itShouldNotifyUser() {
            // When the post form is valid
            PostForm form = new PostForm();
            Validator validator = mock(Validator.class);
            when(validator.validate(form))
                .thenReturn(new HashSet<ConstraintViolation<PostForm>>());

            PostRepository postRepository = mock(PostRepository.class);
            when(postRepository.save(any(JsonObject.class)))
                .thenReturn(UUID.randomUUID().toString());

            PostEditionPresenter presenter
	            = new PostEditionPresenterImpl(validator, postRepository);
            PostEditionView view = mock(PostEditionView.class);

            presenter.save(view, form);

            // Then the application should notify the user
            verify(view).notifySuccess(anyString());
        }

        @Test
        public void itShouldRedirectTheUserToTheShowPostView() {
            // When the form post is valid
            PostForm form = new PostForm();
            Validator validator = mock(Validator.class);
            when(validator.validate(form))
                .thenReturn(new HashSet<ConstraintViolation<PostForm>>());

            PostRepository postRepository = mock(PostRepository.class);
            String aPostId = UUID.randomUUID().toString();
            when(postRepository.save(any(JsonObject.class)))
                .thenReturn(aPostId);

            PostEditionPresenter presenter = new PostEditionPresenterImpl(validator, postRepository);
            PostEditionView view = mock(PostEditionView.class);

            presenter.save(view, form);

            // Then the application should redirect the user
            verify(view).navigateToShowPostView(aPostId);
        }
    }
    
```

In order to execute the scenario the ``view`` must have the following API:

```java
public class PostEditionView {

  /**
     * Display the errors to the user
     */
    public void displayErrors(Set<ConstraintViolation<PostForm>> capture);

    /**
     * Show a success notification message
     */
    public void notifySuccess(String message);

    /**
     * Redirect the user to display a post with the given identifier
     */
    public void navigateToShowPostView(String aPostId);
}

```

And here is the presenter implementation:

```java

public class PostEditionPresenterImpl {

    // ...

	public void save(PostEditionView view, PostForm form) {
        Set<ConstraintViolation<PostForm>> validations = this.validator.validate(form);
        if (!validations.isEmpty()) {
            view.displayErrors(validations);
            return;
        }

        JsonObject json = new JsonObject();
        json.addProperty("title", UUID.randomUUID().toString());
        json.addProperty("content", UUID.randomUUID().toString());
        String postId = this.postRepository.save(json);

        view.notifySuccess("Post edited!");
        view.navigateToShowPostView(postId);
    }

	// ...
}

```

## The view

As previously said the goal is to minimize the responsibilities of the view in order to increase the test coverage of the application. Moreover the thinner the view layer is the easier it is possible the make the view evolve (e.g. framework migration). The responsibilities of the view must be limited to data formatting and binding.

There is no technical constraints on the method names but in order to harmonies things we can use conventions:

* ``bind`` method should be used in order to let the presenter to tell the view to bind the model. So the visibility of this method must be ``public``.

##The presenter
As for the view there is not technical constraints on the method names, only promoted practices.

``initView`` method should be used in order to let the view notify the presenter that it is going to be displayed. If the presenter must provide a model to the view it will pass it by calling ``bind``.

### Boundaries
- Only primitive types  or data objects cross the boundaries from the view to presenter
- Only primitive types or view model cross the boundaries from the presenter to the view

##The model
**The model is not the domain model!** It is a presentation model existing only to holds all necessary information to render the view.

## Notes

All these example have been developed following the TDD methodology.

## Next steps
* View gives access to a navigator object
* View gives access to a notifier object
* View gives access to an eventBus in order to allow inter-view communication


## Resources

* [Supervising Presenter by Martin Fowler](http://martinfowler.com/eaaDev/SupervisingPresenter.html)
* [GUI Architectures by Martin Fowler](http://martinfowler.com/eaaDev/uiArchs.html)
* [Passive View by Martin Fowler](http://martinfowler.com/eaaDev/PassiveScreen.html)
* [Vaadin](https://vaadin.com/home)
* [Sources of the examples](https://github.com/erichonorez/mvp-example)
