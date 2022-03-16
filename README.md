# Game Reviews
#### This is a code summary

This is a django app that allows the user to create, view, and delete reviews on site. The user may also search for top games, games of a specific genre, or search by name. Finally, the user may add, view, and delete favorite games.

User stories were created to determine what functionality should be added, and how that functionality should generally work.

This part was built in an environment with multiple other developers working on other sections of the project.


## User Stories

### Add/View/Edit/Delete
The user should be able to create, view, edit, and delete reviews as mentioned.
Along with this, they have the ability for search for specific reviews, with different data queries.
Finally, we use pagination to allow for a certain amount of items for page.

```buildoutcfg
def add_game(request):
    form = GamesForm(data=request.POST or None)
    if request.method == 'POST':
        if form.is_valid():
            form.save()
            return render(request, 'GameStats/gamestats_home.html')
    context = {'form': form}
    return render(request, 'GameStats/gamestats_create.html', context)

def view_games(request):
    query = request.GET
    # for some reason .all() doesn't work here, instead we will do filter and not include anything *yet*
    # get all and return with no filtering
    game_list = Games.Game.filter()
    if query and request.method == 'GET':
        # filter if we are searching
        if query.get('search_res', None):
            game_list = Games.Game.filter(rating=query.get('search_res', None))
        elif query.get('release_year', None):
            game_list = Games.Game.filter(release_year=query.get('release_year', None))
        elif query.get('genre', None):
            game_list = Games.Game.filter(genre=query.get('genre', None))
        elif query.get('name', None):
            game_list = Games.Game.filter(name=query.get('name', None))
    game = request.GET.get('page', 1)
    paginator = Paginator(game_list, 10)

    try:
        game_list = paginator.page(game)
    except PageNotAnInteger:
        game_list = paginator.page(1)
    except EmptyPage:
        game_list = paginator.page(paginator.num_pages)

    context = { 'game_list': game_list }
    return render(request, 'GameStats/gamestats_all.html', context)


def game_details(request, pk):
    details = get_object_or_404(Games, pk=pk)
    context = {'details': details}
    return render(request, 'GameStats/gamestats_details.html', context)

def game_edit(request, pk):
    details = get_object_or_404(Games, pk=pk)
    form = GamesForm(data=request.POST or None, instance=details)
    if request.method == 'POST':
        if form.is_valid():
            form.save()
            return redirect('gamestats_viewall')
    context = {'form': form}
    return render(request, 'GameStats/gamestats_edit.html', context)

def game_delete(request, pk):
    details = get_object_or_404(Games, pk=pk)
    form = GamesForm(data=request.POST or None, instance=details)
    if request.method == 'POST':
        details.delete()
        return redirect('gamestats_viewall')
    context = { 'details': details, 'form': form }
    return render(request, 'GameStats/gamestats_delete.html', context)
```

### WebScraping

The story indicated the goal of scraping related data from another site.
I chose to do a "top game" list as it made sense.
To make data easy to work with, I made a specific function that would scrape the site and save the data to a dictionary
This way it could be reused when needed, and potentially have parameters when needed.

```buildoutcfg
def scrape_site():
    url = 'https://www.metacritic.com/game'
    user_agent = {'User-agent': 'Mozilla/5.0'}
    response = requests.get(url, headers=user_agent)

    soup = BeautifulSoup(response.text, 'html.parser')
    review_dict = {'name': [], 'date': [], 'rating': [], 'image': [], 'summary': [], 'id': []}

    tempID = 0
    for review in soup.find_all('tr'):
        if review.find("div", class_='clamp-score-wrap'):
            # had to strip everything down because websites add lots of spaces.
            # title
            review_dict['name'].append(review.find("h3").text.strip())
            # score
            review_dict['rating'].append(review.find("div", class_='clamp-score-wrap').text.strip())
            # image URL
            review_dict['image'].append(review.find("img")["src"])
            # summary
            review_dict['summary'].append(review.find(class_="summary").text.strip())

            # release date
            a = review.find_all(class_="clamp-details")
            for element in a:
                try:
                    # in the event we come up with something other than a normal string, .text.strip() won't work
                    # thus cleaning invalid inputs, and we just don't add to the dict if they are invalid
                    review_dict['date'].append(element.find("span").text.strip())
                except AttributeError:
                    pass
            review_dict['id'].append(tempID)
            tempID += 1
    return review_dict
```

Next up I added genre scraping from the same site.
Ironically I did a separate function here.
Although the format is similar, doing a separate function made since due to the differences in the data.

```buildoutcfg
def scrape_genre(genre):
    url = "https://www.metacritic.com/browse/games/genre/date/" + genre + "/all"
    user_agent = {'User-agent': 'Mozilla/5.0'}
    response = requests.get(url, headers=user_agent)
    review_dict = {'name': [], 'platform': [], 'rating': []}
    soup = BeautifulSoup(response.text, 'html.parser')

    for review in soup.find_all('tr'):
        if review.find("div", class_='clamp-score-wrap'):
            # had to strip everything down because websites add lots of spaces.
            # title
            review_dict['name'].append(review.find("h3").text.strip())
            # print(review.find("h3").text.strip())
            # platform
            review_dict['platform'].append(review.find("div", class_='platform').text.strip())

            # rating
            a = review.find_all(class_="clamp-details")
            review.find(class_='large')
            review_dict['rating'].append(review.find(class_='metascore_anchor').find(class_='game').text.strip())
    return review_dict


def view_genre(request, genre):
    review_dict = scrape_genre(genre=genre)
    name = review_dict['name'][:10]
    date = review_dict['platform'][:10]
    rating = review_dict['rating'][:10]

    test = zip(name, date, rating)


    context = { 'data': test, 'genre': str(genre).capitalize() }
    return render(request, 'GameStats/gamestats_genres.html', context)
```

#### Favorites
##### Roping this into the webscraping portion as the implementation mostly used the webscraping portion
The use should be able to add, view and remove favorites.
This is great, but I wanted to take it a step further
I added the normal add favorite and remove favorite area, but no one really wants to seek out those pages
In the specific game view, where it gives a rating, details, and summary, I added the ability to fav/unfav games.

```buildoutcfg
def top_game_one(request, id):
    review_dict = scrape_site()
    favorited = check_fav_duplicate(review_dict['name'][id])
    pk = get_fav_id(review_dict['name'][id])
    if request.method == 'POST':
        if favorited:
            favorite_delete(pk)
        else:
            try_add_favorite(review_dict['name'][id], review_dict['rating'][id], review_dict['date'][id])
        favorited = not favorited
    context = {'name': review_dict['name'][id], 'date': review_dict['date'][id], 'rating': review_dict['rating'][id],
               'image': review_dict['image'][id], 'summary': review_dict['summary'][id], 'status': favorited, 'id': pk }
    return render(request, 'GameStats/gamestats_view_one.html', context)
```
The code is pretty self-explanatory, when loading the page it will check for the status of the favorite
It will pass the data to the webpage, and since the button state is boolean, we just do a flipflop on the variable and reload the page.
With a little more effort, I believe I could have used ajax rather than a full page reload.

### API queries

This leaves us with APIs
I added basic genre functionality to the webscraping because API searching was awful
As terrible as it is, I got it mostly working, just with missing data due to limitations of the datasource
I also followed a similar pattern, had a general function that does a query
Some other limitations that followed were data structures. To get multiple items, I was missing data such as the genre.
To get around this, I did the first query, then I did a query for each of those and pulled the data I needed
This of course caused more queries than we'd like to do, but it functioned as necessary.

```buildoutcfg
def api_game_view(request):
    query = request.GET
    query_options = ""
    if query and request.method == 'GET':
        if query.get('name', None):
            query_options = '&query={}'.format(query.get('name', None))

    filtered_responses = api_query(query_options)

    for i in range(len(filtered_responses['genres'])):
        filtered_responses['genres'][i] = str(filtered_responses['genres'][i]).strip("[]").replace("'", "")

    dataList = zip(filtered_responses['name'], filtered_responses['genres'], filtered_responses['release_date'])
    context = {'data': dataList}

    return render(request, 'GameStats/gamestats_explore.html', context)

def api_query(options=None):
    user_agent = {'User-agent': 'Mozilla/5.0'}
    req_url = "https://www.giantbomb.com/api/games/?api_key=8c2a3059218223501315304c270747790b292c62"
    if options:
        req_url = "http://www.giantbomb.com/api/search?api_key=8c2a3059218223501315304c270747790b292c62"
    # limit 5 for testing purposes.
    # set to 10 for main integration
    filters = "&limit=10&format=json"
    filters += options
    req = requests.get(req_url + filters, headers=user_agent)
    filtered_responses = {'name': [], 'genres': [], 'release_date': []}
    try:
        response_obj = json.loads(req.text)
        # generally works but doesn't always have an actual release date
        # the amount of data and the way they sort it is ridiculous
        for i in range(len(response_obj['results'])):
            # each specific piece of data has its potential for an exception
            try:
                # if this tosses an exception we don't want to add any other data
                # otherwise it will misalign names with incorrect data
                # so skip it and the rest of this loop, on to the next iteration
                filtered_responses['name'].append(response_obj['results'][i]['name'])
            except:
                continue
            try:
                # if this one errors, none were found and we just need to list that
                # limitation of the api data.
                filtered_responses['genres'].append(get_api_genres(i, response_obj))
            except:
                filtered_responses['genres'].append("None found")
            try:
                # if this one errors, none were found and we just need to list that
                # limitation of the api data.
                filtered_responses['release_date'].append(response_obj['results'][i]['original_release_date'])
            except:
                filtered_responses['release_date'].append(response_obj['results'][i]['expected_release_year'])
    except Exception as e:
        # not a necessary print, sometimes there is a json error though for bad data
        print(e)
```



## Final thoughts/About
Ultimately with more time, things could be ironed out, done better, but I am confident that this displays my knowledge
of django and certain coding aspects in general.

During the course of this project, I was able to work alongside others in the same code base.
This allowed me to properly utilize git to merge, branch, fetch, and create pull requests without messing up the main branch.  All git management was done utilizing Azure DevOps.

Finally, this project was worked on in two one-week sprints.  We utilized scrum throughout the entire process.

##### Build details:
Python 3.5
</br>Django 3
