---
layout: post
title:  "Attributed String with Style"
author: "Pierre Felgines"
---

Have you ever had the feeling that your `NSAttributedString` were not in the right place in your code? That you were mixing view details and data logic?
It happened to me recently and I want to propose a solution that focus separately on the **semantics** and on the **visual representation** of an attributed string.

## The Problem

Apple defines an `NSAttributedString` as
> A string that has associated attributes (such as visual style, hyperlinks, or accessibility data) for portions of its text.

In general, I use `NSAttributedString` to highlight parts of a label content (with bold font, underline, etc.).
For instance, for one of the last app I worked on, I had to be able to search through a list of users. As I type in the search bar, users are filtered based on the query, and their names contain a visual indication of the query match. This visual clue is implemented with an `NSAttributedString`.

{% include image.html
            img="assets/attributed_string_example.png"
            title="Image 1. Highlighted search term"
            caption="Image 1. Highlighted search term" %}

Let's see the relevant parts of the implementation to understand what is wrong.

The view is a subclass of `UITableViewCell`:

{% highlight swift %}
class UserTableViewCell: UITableViewCell {

    @IBOutlet private var userImageView: AvatarImageView!
    @IBOutlet private var nameLabel: UILabel!
    @IBOutlet private var typeLabel: UILabel!

    override func awakeFromNib() {
        super.awakeFromNib()
        setup()
    }

    private func setup() {
        typeLabel.font = UIFont.systemFont(ofSize: 13, weight: .regular)
        typeLabel.textColor = UIColor.ad_defaultSubtitle
    }

    func configure(with viewModel: UserTableViewCellViewModel) {
        userImageView.sd_setImage(with: viewModel.imageURL)
        nameLabel.attributedText = viewModel.attributedName
        typeLabel.text = viewModel.type
    }
}
{% endhighlight %}

The cell is configured with a *ViewModel*, a dumb struct that holds the data to display.

{% highlight swift %}
struct UserTableViewCellViewModel {
    let imageURL: URL
    let attributedName: NSAttributedString
    let type: String
}
{% endhighlight %}

Note that the `typeLabel` visual attributes are defined in the view, but not the `nameLabel` ones, that live in the attributed string.

The glue code between the view and the view model is a *Mapper*: an object that takes an entity (in our case a `User`), and returns a view model to display. This is where the attributed string is created.

{% highlight swift %}
struct UserTableViewCellViewModelMapper {

    let user: User
    let searchText: String?

    func map() -> UserTableViewCellViewModel {
        let attributedName = attributedUserName(
            from: user.name,
       	    searchText: searchText
        )
        return UserTableViewCellViewModel(
            imageURL: user.avatarUrl,
            attributedName: attributedName,
            type: user.type
        )
    }

    // MARK: - Private

    private func attributedUserName(from userName: String, searchText: String?) -> NSAttributedString {
        let regularAttributes: [NSAttributedString.Key: Any] = [
            .font: UIFont.systemFont(ofSize: 14.0)
        ]
        let boldAttributes: [NSAttributedString.Key: Any] = [
            .font: UIFont.boldSystemFont(ofSize: 14.0)
        ]
        let attributedUserName = NSMutableAttributedString(
            string: userName,
            attributes: regularAttributes
        )
        if let searchText = searchText {
            let range = NSString(string: userName).range(of: searchText)
            attributedUserName.addAttributes(boldAttributes, range: range)
        }
        return attributedUserName
    }
}
{% endhighlight %}

This works pretty well, but a detail was bothering me. I do not want to set up the visual style of my strings in a mapper. The mapper's only job is to convert model data to view data. It should not be aware of the way the data is displayed to the end user.

But at the same time, the mapper needs to know how to format the data, and in our case, which part of the username is highlighted. And I don't want my view to know about the mapping operation. The view is dumb and should only display whatever data we pass in.

## The solution

The solution is to split the concerns. We need to create **semantic styles**. These styles will be used on the attributed string in the mapper. Then the view will associate a style to visual attributes (color, font, etc.). This way the mapper only knows about the semantics, and the view about the display. It's the same idea behind HTML and CSS, the html should be a bunch of semantic data and CSS a list of rules to style the data.

The mapper now becomes:

{% highlight swift %}
struct UserTableViewCellViewModelMapper {

   ...

    private func attributedUserName(from userName: String, searchText: String?) -> NSAttributedString {
    	let regularAttributes: [NSAttributedString.Key: Any] = [
            .semanticStyle: UserTableViewCellViewModel.Style.regular
        ]
        let highlightedAttributes: [NSAttributedString.Key: Any] = [
            .semanticStyle: UserTableViewCellViewModel.Style.searchHighlighted
        ]
        let attributedUserName = NSMutableAttributedString(
            string: userName,
            attributes: regularAttributes
        )
        if let searchText = searchText {
            let range = NSString(string: userName).range(of: searchText)
            attributedUserName.addAttributes(boldAttributes, range: range)
        }
        return attributedUserName
    }
}
{% endhighlight %}

Note that I don't use `.font` attribute anymore but a new custom `.semanticStyle` property that is completely independent of `UIKit`.

The styles, that can be anything, are in my case defined in the view model.

{% highlight swift %}
struct UserTableViewCellViewModel {

    enum Style {
        case regular
        case searchHighlighted
    }

    ...
}
{% endhighlight %}

Now in the view, I can choose which `UIKit` attributes are associated to each style.

{% highlight swift %}
class UserTableViewCell: UITableViewCell {

	...

    private lazy var styler: AttributedStringStyler<UserTableViewCellViewModel.Style> = createStyler()

    func configure(with viewModel: UserTableViewCellViewModel) {
        ...
        nameLabel.attributedText = viewModel.attributedName.styled(with: styler)
    }

    // MARK: - Private

    private func createStyler() -> AttributedStringStyler<UserTableViewCellViewModel.Style> {
        let styler = AttributedStringStyler<UserTableViewCellViewModel.Style>()
        styler.register(
            attributes: [
                .font: UIFont.systemFont(ofSize: 14.0),
            ],
            forStyle: UserTableViewCellViewModel.Style.regular
        )
        styler.register(
            attributes: [
                .font: UIFont.boldSystemFont(ofSize: 14.0)
            ],
            forStyle: UserTableViewCellViewModel.Style.searchHighlighted
        )
        return styler
    }
}
{% endhighlight %}

If you wonder, the `styler` is rather simple. It just enumerates all the attributes and replace the `semanticStyle` occurrences with registered attributes for the current style. You can find the implementation on my [Github repository](https://github.com/felginep/AttributedStringStyle).

## Final note

Now the code feels right and the concerns are respected. For sure, it adds a layer of complexity, but it clearly separates the view and the mapping logic. On one side we focus on the semantics, on the other side on the display.
That means the mapper does not have to evolve if the design change in the future. Conversely, the view will not change its attributes if the backing data is updated (for instance, if we modify the highlighting feature).








